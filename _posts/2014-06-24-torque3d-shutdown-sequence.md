---
layout: post
title:  "The Torque 3D shutdown sequence - a safari"
date:   2014-06-24 20:45
tags:   programming c++ torque safaris
---

Welcome to the second Torque 3D safari. In this series I document my explorations
in the [Torque 3D][] source code. Today, I'm exploring the engine's shutdown
sequence in order to change the engine's [exit status][] code. Currently, it
always returns 0 (success). For the purpose of unit testing, I'd like to see if
I can make it return a 1 (or any other code) if I want it to (i.e., if tests
fail).

[Torque 3D]: http://torque3d.org
[exit status]: http://en.wikipedia.org/wiki/Exit_status

## Picking up the trail

We'll start by searching for the `quit` console function, which is what we call
from scripts to shut down the engine. Doing a text search in the engine yields a
ton of lines with `quit`, but only one that's a `DefineConsoleFunction`. Its body
is pretty simple:

    Platform::postQuitMessage(0);

Okay, well let's try changing that `0` to a `1`. No dice, we still get a `0` code
on return. Let's follow the trail. `Platform::postQuitMessage` is also pretty
simple, and just does this:

    PostQuitMessage(in_quitVal);

...and that's a Windows API function. Huh. That's going nowhere. So it looks like
we'll have to look at this from the other end - figure out what causes the engine's
main loop to stop, and catch the problem from that end.

## The main loop

I happen to already know that the engine main loop lives in `StandardMainLoop::doMainLoop`,
but that's too far in to be useful yet. What we need to find is where the executable
returns and work back from there. So we'll start in the game project (not the DLL
project) and open `main.cpp`. `WinMain` is our target.

    torque_winmain = (...cast...)GetProcAddress(hGame, "torque_winmain");
	 ...
    int ret = torque_winmain(hInstance, hPrevInstance, lpszCmdLine, nCommandShow);
	 ...
    return ret;

I've picked out the pertinent lines. Basically, what's happening here is that
the Torque DLL is loaded, then we pull out a function called `torque_winmain`,
run it, and return what it gives us. Let's find it.

Back in the DLL project, I do a text search for `torque_winmain` and find this in
`platformWin32/winWindow.cpp`:

    S32 torque_winmain( HINSTANCE hInstance, HINSTANCE, LPSTR lpszCmdLine, S32)

Which does this:


    S32 retVal = TorqueMain(argv.size(), (const char **) argv.address());
    ...
    return retVal;

Okay, let's follow the rabbit hole to the definition of `TorqueMain`. I will note
at this point that all these functions are returning `S32`s, signed 32-bit
integers, which bodes well. `TorqueMain` is pretty simple:

    S32 TorqueMain(int argc, const char **argv)
    {
       if (!torque_engineinit(argc, argv))
          return 1;

       while(torque_enginetick()) {}

       torque_engineshutdown();
       return 0;
    }

Well there's your problem, then. So where can we get a return code from to
provide here? `torque_enginetick` does return an `S32`, but let's see where it
gets that number from.

    bool ret = StandardMainLoop::doMainLoop(); 
    return ret;

Right. So here we're getting a `bool` from `doMainLoop` and casting it up to
an `S32`. Which evidently is `true`, or `1`, until the engine exits. So it looks
like we do some to `doMainLoop` after all. Which looks kind of like:

    bool StandardMainLoop::doMainLoop()
    {
       bool keepRunning = true;
		 ...

       if(!Process::processEvents())
          keepRunning = false;
	    ...

       return keepRunning;
    }

Right. So `doMainLoop` always returns `true`, until we shut the engine down. So
how does the quit signal cause the main loop to stop, and how could we get that
value out of the main loop?

## Event processing

I guess we'll dive into `processEvents`.

    bool Process::processEvents()
    {
       Process::get()._signalProcess.trigger();

       if (!Process::get()._RequestShutdown) 
          return true;

       Process::get()._RequestShutdown = false;
       return false;
    }

Okay, so if if `_RequestShutdown` is `false`, we'll return `true` (i.e., keep
running). Otherwise we'll tell the main loop to stop. I'm really not sure what
this `Process` machinery does, but I'm interested in this `trigger` method.
Let's first see if we can see where `_RequestShutdown` might be set. I do a 'find
all references' and immediately turn up:

    void Process::requestShutdown()
    {
       Process::get()._RequestShutdown = true;
    }

Well that was less mysterious than I expected. Let's see where this is called.
Turns out it actually happens a lot. One in `GuiCanvas` handling `WindowClose`
and `WindowDestroy` events, and some in unit tests for... networking? Okay, here's
a unit test for `Process` which may shed some light on the matter.

    void process()
    {
       if(_remainingTicks==0)
          Process::requestShutdown();
 
       _remainingTicks--;
    }

    void run()
    {
       // We'll run 30 ticks, then quit.
       _remainingTicks = 30;
 
       // Register with the process list.
       Process::notify(this, &TestingProcess::process);
 
       // And do 30 notifies, making sure we end on the 30th.
       for(S32 i=0; i<30; i++)
          test(Process::processEvents(), "Should quit after 30 ProcessEvents() calls - not before!");
       test(!Process::processEvents(), "Should quit after the 30th ProcessEvent() call!");
 
       Process::remove(this, &TestingProcess::process);
    }

Okay, well... that makes sense. So every process tick, the `process` method of
the unit test class will be called, and it will shut down when `30` ticks have
passed. Which causes `processEvents` to return `false`. I think.

Um. That's kind of what we knew already. Just more condensed. Okay, cool.

So `Process::notify` is important to this somehow. But let's actually forget about
it and see if we can go even deeper. The only two likely candidates left that
aren't unit tests are two results from `windowManager/win32/winDispatch.cpp`.
One is wrapped inside `Journal::isPlaying`, which doesn't sound relevant, and
the other is:

    	case WM_QUIT: {
    	   // Quit indicates that we're not going to receive anymore Win32 messages.
    	   // Therefore, it's appropriate to flag our event loop for exit as well,
    	   // since we won't be getting any more messages.
    	   Process::requestShutdown();
    	   break;
    	          }

And `WM_QUIT` is a Windows API event. I'm going to infer that this is as close
to closure as we'll get. But let's explore this a bit. This code is from
`_dispatch`, which is... well, let's see how it's used. It's called in two places,
both in that same file (`winDispatch.cpp`). One is in `Dispatch`, and one in
`DispatchNext`. These are short, so let's check them out:


    // Dispatch the window event, or queue up for later
    void Dispatch(DispatchType type,HWND hWnd,UINT message,WPARAM wparam,WPARAM lparam)
    {
       // If the message queue is not empty, then we'll need to delay
       // this dispatch in order to preserve message order.
       if (type == DelayedDispatch || !_MessageQueue.isEmpty())
          _MessageQueue.post(hWnd,message,wparam,lparam);
       else
          _dispatch(hWnd,message,wparam,lparam);
    }
    
    // Dispatch next even in the queue
    bool DispatchNext()
    {
       WinMessageQueue::Message msg;
       if (!_MessageQueue.next(&msg))
          return false;
       _dispatch(msg.hWnd,msg.message,msg.wparam,msg.lparam);
       return true;
    }

So what this looks like to me is that incoming window events go to `Dispatch`,
where they either get processed immeidately or must wait in a queue. Then I guess
`DispatchNext` is called periodically to consume the queue. Let's see where
`Dispatch` is called from... oh shoot, that's lots of calls. Let's not go there.
Let's look for `DispatchNext` instead. Ooh, it's only called in one location,
in `win32WindowManager.cpp`.

Atually, let's not follow this particular rabbit-hole. I think we've learned as
much as we can from the engine source. It's time to see if we can follow the
trail through Windows API-land.

## The Windows API

Google `PostQuitMessage` from all the way back up there. Find [this][winapi]
helpful page. Most pertinently:

> _nExitCode_ [in]
> 
> Type: _int_
> 
> The application exit code. This value is used as the wParam parameter of the
> WM_QUIT message.

Okay, so Windows API messages have a parameter. Okay, I'm glad we looked up
`Dispatch` and `DispatchNext` above, because of this:

    void Dispatch(..., WPARAM wparam, ...)

Ah. So where does that value go when we receive it? Let's jump back into `_dispatch`
where we act on the `WM_QUIT` event by calling `Platform::requestShutdown`. And...

    static bool _dispatch(..., WPARAM wParam, ...)

Bingo. So anywhere in `_dispatch` we can make use of `wParam`, which seems to
have type `UINT_PTR`... which sounds like a pointer to an unsigned integral type, 
but in any case, we can use it to access the exit status code.

That's where I'll leave this safari, as the rest is mechanically adding more code
to store and make use of the status code.

[winapi]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms644945(v=vs.85).aspx
