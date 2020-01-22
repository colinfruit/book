# Implementing a Syscall Interface for Userspace

This guide provides an overview and walkthrough on how to add a syscall
interface for userspace applications in Tock. This syscall interface exposes
some kernel functionality to applications. For example, this could be the
ability to sample a new sensor, or use some service like doing AES encryption.

In this guide we will use a running example of providing a userspace interface
for a hypothetical water level sensor (the "WS00123" water level sensor). This
interface will allow applications to query the current water level, as well as
get notified when the water level exceeds a certain threshold.

## Setup

This guide assumes you already have existing kernel code that needs a
userspace interface. Likely that means there is already a capsule implemented.
Please see the other guides if you also need to implement the capsule.

We will assume there is a `struct WS00123 {...}` object already implemented
that includes all of the logic needed to interface with this particular water sensor.

## Overview

The high-level steps required are:

1. Decide on the interface to expose to userspace.
2. Map the interface to the existing syscalls in Tock.
3. Create grant space for the application.
4. Implement the `Driver` trait.
5. Document the interface.
6. Expose the interface to userspace.
7. Implement the syscall library in userspace.

## Step-by-Step Guide

The steps from the overview are elaborated on here.

1. **Decide on the interface to expose to userspace.**

    Creating the interface for userspace means making design decisions on how
    applications should be able to interface with the kernel capsule. This can
    have a lasting impact, and is worth spending some time on up-front to avoid
    implementing an interface that is difficult to use or does not match the
    needs of applications.

    While there is not a fixed algorithm on how to create such an interface,
    there are a couple tips that can help with creating the interface:

    - Consider the interface for the same or similar functionality in other
      systems (e.g. Linux, Contiki, TinyOS, RIOT, etc.). These may have iterated
      on the design and include useful features.
    - Ignore the specific details of the capsule that exists or how the
      particular sensor the syscall interface is for works, and instead consider
      what a user of that capsule might want. That is, if you were writing an
      application, how would you expect to use the interface? This might be
      different from how the sensor or other hardware exposes features.
    - Consider other chips that provide similar functionality to the specific
      one you have. For example, imagine there is a competing water level sensor
      the "OWlS789". What features do both provide? How would a single interface
      be usable if a hardware board swapped one out for the other?

    The interface should include both actions (called "commands" in Tock) that
    the application can take (for example, "sample this sensor now"), as well as
    events (called subscribe callbacks in Tock) that the kernel can trigger
    inside of an application (for example, when the sensed value is ready).

    The interface can also include memory sharing between the application and
    the kernel. For example, if the application wants to receive a number of
    samples at once, or if the kernel needs to operate on many bytes (say for
    example encrypting a buffer), then the interface should allow the
    application to share some of its memory with the kernel to enable that
    functionality.

2. **Map the interface to the existing syscalls in Tock.**

    With a sketch of the interface created, the next step is to map that
    interface to the specific syscalls that the Tock kernel supports. Tock has
    three main relevant syscall operations that applications can use when
    interfacing with the kernel:

    1. `allow`: This lets an application share some of its memory with the
       kernel.

    2. `subscribe`: This provides a callback function pointer that the kernel
       can use to trigger a callback in the application.

    3. `command`: This enables the application to direct the kernel to do some
       action.

    All three also include a couple other parameters to differentiate different
    commands, subscriptions, or allows. Refer to the more detailed documentation
    on the Tock syscalls for more information.

    As the Tock kernel only supports these syscalls, each feature in the design
    you created in the first step must be mapped to one or more of these
    syscalls. To help, consider these hypothetical interfaces that
    an application might have for our water sensor:

    - _What is the maximum water level?_ This can be a simple command, where the
      return value of the command is the maximum water level.
    - _What is the current water level?_ This will require two steps. First,
      there needs to be a subscribe call where the application can setup a
      callback function. The kernel will call this when the water level value
      has been acquired. Second, there will need to be a command to instruct the
      kernel to take the water level reading.
    - _Take ten water level samples._ This will require three steps. First, the
      application must use an allow syscall to share a buffer with the kernel
      large enough to hold 10 water level readings. Then it must setup a
      subscribe callback that the kernel will call when the 10 readings are
      ready (note this callback function can be the same as in the single sample
      case). Finally it will use a command to tell the kernel to start sampling.
    - _Notify me when the water level exceeds a threshold._ A likely way to
      implement this would be to first require a subscribe syscall for the
      application to set the function that will get called when the high water
      level event occurs. Then the application will need to use a command to
      enable the high water level detection and to optionally set the threshold.

    **Checkpoint**: You have defined how many allow, subscribe, and command
    syscalls you need, and what each will do.


3. **Create grant space for the application.**

    Grants are regions in a process's memory space that are shared with the
    kernel. The kernel uses these to store state on behalf of the process. To
    provide our syscall interface for the water level sensor, we need to setup a
    grant so that we can store state for all of the requests we may get from
    processes that want to use the sensor. In particular, we will need to be
    able to store the callback that a process registers with us.

    The first step to do this is to create a struct that contains fields for all
    of the state we want to store for each process that uses our syscall interface. By convention in Tock, this struct is named `App`, but it could
    have a different name. We will need to keep two values, the callback and the
    high water alert threshold:

    ```rust
    pub struct App {
        callback: Option<Callback>,
        threshold: usize,
    }
    ```

    Now that we have the type we want to store in the grant region we can create
    the grant type for it by extending our `WS00123` struct:

    ```rust
    pub struct WS00123 {
    	...
        apps: Grant<App>,
    }
    ```

    We will also need the grant region to be created by the board and passed in
    to us by adding it to the capsules `new()` function:

    ```rust
    impl WS00123 {
        pub fn new(
            ...
            grant: Grant<App>,
        ) -> WS00123 {
            WS00123 {
                ...,
                apps: grant,
            }
        }
    }
    ```

    Now we have somewhere to store values on a per-process basis.

4. **Implement the `Driver` trait.**

    The `Driver` trait is how a capsule provides implementations for the various
    syscalls an application might call. The basic framework looks like:

    ```rust
    impl Driver for WS00123 {
    	fn allow(
    	    &self,
    	    appid: AppId,
    	    allow_num: usize,
    	    slice: Option<AppSlice<Shared, u8>>,
    	) -> ReturnCode { }

        fn subscribe(
            &self,
            subscribe_num: usize,
            callback: Option<Callback>,
            _app_id: AppId,
        ) -> ReturnCode { }

        fn command(&self,
        	       command_num: usize,
        	       data: usize,
        	       data2: usize,
        	       app: AppId,
        ) -> ReturnCode { }
    }
    ```

    Note: there are default implementations for each of these, so in our water
    level sensor case we can simply omit the `allow` call.

    By Tock convention, every syscall interface must at least support the
    command call with `command_num == 0`. This allows applications to check if
    the syscall interface is supported on the current platform. If the command
    returns `<0` then the syscall interface is not present. Many implementations
    return `SUCCESS` (i.e. 0), however, some return other positive numbers, like
    for example the number of LEDs present on the board. For our example, we use
    the simple case:

    ```rust
    impl Driver for WS00123 {
    	fn command(&self,
        	       command_num: usize,
        	       data: usize,
        	       data2: usize,
        	       app: AppId,
        ) -> ReturnCode {
    		match command_num {
    			0 => ReturnCode::SUCCESS,
    			_ => ReturnCode::ENOSUPPORT,
    		}
        }
    }
    ```

    Now let's handle the subscribe call where the app can setup the callback we
    should use. For this capsule, we will use a single callback for both when a
    measurement is ready and for when a high water alert is triggered, but with
    different arguments passed into the callback.

    ```rust
    impl Driver for WS00123 {
        /// Setup callbacks.
        ///
        /// ### `subscribe_num`
        ///
        /// - `0`: Setup the main callback to be used when samples are ready
        ///        and when any alerts are triggered.
        fn subscribe(
            &self,
            subscribe_num: usize,
            callback: Option<Callback>,
            app_id: AppId,
        ) -> ReturnCode {
            self.apps
                .enter(app_id, |app, _| {
                    match subscribe_num {
                        0 => app.callback = callback,
                        _ => return ReturnCode::ENOSUPPORT,
                    }
                    ReturnCode::SUCCESS
                })
                .unwrap_or_else(|err| err.into())
        }
    }
    ```

    As you can see, we use the `enter()` function to "enter" the grant region of
    the specific requesting app `app_id`. This performs checks like ensuring the
    grant region exists and that the application is valid. If `enter()` succeeds
    then we can update the `App` state like normal. Here we only need to save
    the callback.

    Next we can implement more commands so that the application can direct our
    capsule as to what the application wants us to do. We need two commands, one
    to sample and one to enable the alert. In both cases the commands must
    return a `ReturnCode`, and call functions that likely already exist in the
    original implementation of the `WS00123` sensor. If the functions don't
    quite exist, then they will need to be added as well.

    ```rust
    impl Driver for WS00123 {
    	/// Command interface.
    	///
    	/// ### `command_num`
    	///
    	/// - `0`: Return SUCCESS if this driver is included on the platform.
    	/// - `1`: Start a water level measurement.
    	/// - `2`: Enable the water level detection alert. `data` is used as the
    	///        height to set as the the threshold for detection.
    	fn command(&self,
        	       command_num: usize,
        	       data: usize,
        	       data2: usize,
        	       app: AppId,
        ) -> ReturnCode {
    		match command_num {
    			0 => ReturnCode::SUCCESS,

    			1 => self.start_measurement(app),

    			2 => {
    				// Save the threshold for this app.
    				self.apps
    				    .enter(app_id, |app, _| {
    				        app.threshold = data;
    				        ReturnCode::SUCCESS
    				    })
    				    .map_or_else(
    				    	|err| err.into(),
    				    	|ok| self.set_high_level_detection()
    				    )
    			}

    			_ => ReturnCode::ENOSUPPORT,
    		}
        }
    }
    ```

    The last item that needs to be added is to actually use the callback when
    the sensor has been sampled or the alert has been triggered. Actually
    issuing the callback will need to be added to the existing implementation of
    the capsule. As an example, if our water sensor was attached to the board
    over I2C, then we might trigger the callback in response to a finished I2C
    command:

    ```rust
    impl i2c::I2CClient for WS00123 {
        fn command_complete(&self, buffer: &'static mut [u8], _error: i2c::Error) {
        	...
        	let app_id = <get saved appid for the app that issued the command>;
        	let measurement = <calculate water level based on returned I2C data>;

        	self.apps.enter(app_id, |app, _| {
        	    app.callback.map(|mut cb| cb.schedule(0, measurement, 0));
        	});
        }
    }
    ```

    There may be other cleanup code required to reset state or prepare the
    sensor for another sample by a different application, but these are the
    essential elements for implementing the syscall interface.

    Finally, we need to assign our new `Driver` implementation a number so that
    the kernel (and userspace apps) can differentiate this syscall interface
    from all others that a board supports. By convention this is specified by
    a global value at the top of the capsule file:

    ```rust
    pub const DRIVER_NUM: usize = 0x80000A;
    ```

    The value cannot conflict with other capsules in use, but can be set
    arbitrarily, particularly for testing. Tock has a procedure for assigning
    numbers, and you may need to change this number if the capsule is to merged
    into the main Tock repository.

    **Checkpoint**: You have the syscall interface translated from a design to
    code that can run inside the Tock kernel.


5. **Document the interface.**

    A syscall interface is a contract between the kernel and any number of
    userspace processes, and processes should be able to be developed
    independently of the kernel. Therefore, it is helpful to document the new
    syscall interface you made so applications know how to use the various
    command, subscribe, and allow calls.

    An example markdown file documenting our water level syscall interface is
    as follows:

    ```md
    ---
    driver number: 0x80000A
    ---

    # Water Level Sensor WS00123

    ## Overview

    The WS00123 water level sensor can sample the depth of water as well as
    trigger an event if the water level gets too high.

    ## Command

      * ### Command number: `0`

        **Description**: Does the driver exist?

        **Argument 1**: unused

        **Argument 2**: unused

        **Returns**: SUCCESS if it exists, otherwise ENODEVICE

      * ### Command number: `1`

        **Description**: Initiate a sensor reading.  When a reading is ready, a
        callback will be delivered if the process has `subscribed`.

        **Argument 1**: unused

        **Argument 2**: unused

        **Returns**: `EBUSY` if a reading is already pending, `ENOMEM` if there
        isn't sufficient grant memory available, or `SUCCESS` if the sensor
        reading was initiated successfully.

      * ### Command number: `2`

        **Description**: Enable the high water detection. THe callback will the
        alert will be delivered if the process has `subscribed`.

        **Argument 1**: The water depth to alert for.

        **Argument 2**: unused

        **Returns**: `EBUSY` if a reading is already pending, `ENOMEM` if there
        isn't sufficient grant memory available, or `SUCCESS` if the sensor
        reading was initiated successfully.

    ## Subscribe

      * ### Subscribe number: `0`

        **Description**: Subscribe a callback for sensor readings and alerts.

        **Callback signature**: The callback's first argument is `0` if this is
        a measurement, and `1` if the callback is an alert. If it is a
        measurement the second value will be the water level.

        **Returns**: SUCCESS if the subscribe was successful or ENOMEM if the
        driver failed to allocate memory to store the callback.
    ```

    This file should be named `<driver_num>_<sensor>.md`, or in this case:
    `80000A_ws00123.md`.


6. **Expose the interface to userspace.**

    The last kernel implementation step is to let the main kernel know about
    this new syscall interface so that if an application tries to use it the
    kernel knows which implementation of `Driver` to call. In each board's
    `main.rs` file (e.g. `boards/hail/src/main.rs`) there is a implementation of
    the `Platform` trait where the board can setup which syscall interfaces it
    supports. To enable our water sensor interface we add a new entry to the
    match statement there:

    ```rust
    impl Platform for Hail {
        fn with_driver<F, R>(&self, driver_num: usize, f: F) -> R
        where
            F: FnOnce(Option<&dyn kernel::Driver>) -> R,
        {
            match driver_num {
            	...
                capsules::ws00123::DRIVER_NUM => f(Some(self.ws00123)),
                ...
                _ => f(None),
            }
        }
    }
    ```


7. **Implement the syscall library in userspace.**

    At this point userspace applications can use our new syscall interface and
    interact with the water sensor. However, applications would have to call all
    of the syscalls directly, and that is fairly difficult to get right and not
    user friendly. Therefore, we typically implement a small library layer in
    userspace to make using the interface easier.

    In this guide we will be setting up a C library, and to do so we will create
    `libtock-c/libtock/ws00123.h` and `libtock-c/libtock/ws00123.c`, both of
    which will be added to the libtock-c repository. The .h file defines the
    public interface and constants:

    ```c
	#pragma once

	#include "tock.h"

	#ifdef __cplusplus
	extern "C" {
	#endif

	#define DRIVER_NUM_WS00123 0x80000A

	int ws00123_set_callback(subscribe_cb callback, void* callback_args);
	int ws00123_read_water_level();
	int ws00123_enable_alerts(uint32_t threshold);

	#ifdef __cplusplus
	}
	#endif
	```

	While the .c file provides the implementations:

	```c
	#include "ws00123.h"
	#include "tock.h"

	int ws00123_set_callback(subscribe_cb callback, void* callback_args) {
	  return subscribe(DRIVER_NUM_WS00123, 0, callback, callback_args);
	}

	int ws00123_read_water_level() {
	  return command(DRIVER_NUM_WS00123, 1, 0, 0);
	}

	int ws00123_enable_alerts(uint32_t threshold) {
	  return command(DRIVER_NUM_WS00123, 2, threshold, 0);
	}
	```

	This is a very basic implementation of the interface, but it provides some
	more readable names to the numbers that make up the syscall interface. See
	other examples in libtock for how to make synchronous versions of
	asynchronous operations (like reading the sensor).


## Wrap-Up

Congratulations! You have added a new API for userspace applications using the
Tock syscall interface! We encourage you to submit a pull request to upstream
this to the Tock repository.