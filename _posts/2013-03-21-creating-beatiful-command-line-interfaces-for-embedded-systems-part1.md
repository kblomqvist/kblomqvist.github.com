---
layout: post
title: "Creating beautiful command-line interfaces for embedded systems&ndash;Part I: initial efforts"
description: "Testing docopt.c with Aery32"
tweet-text: "Creating beautiful command-line interfaces for embedded systems&ndash;Part I: initial efforts"
author: Kim Blomqvist
---

Ever since I managed to send my first character from my PC to
a microcontroller, I've had an itch to implement a beautiful
CLI for all of my hardware. So far I've settled to using single
character commands followed by positional arguments, which I parse
with `sscanf()` like this

<pre class="prettyprint lang-c">
getline(line);
switch(line[0]) {
    case 'a':
        sscanf(&line[1], "%d %d", &a_arg1, &a_arg2);
        run_command_a(a_arg1, a_arg2);
        break;
    case 'b':
        sscanf(&line[1], "%u", &b_arg1);
        run_command_b(b_arg1);
        break;
    default:
        printf("Unkown command '%c'", line[0]);
}
</pre>

Although this works very well, I still have missed the word-length commands with
possible subcommands and options. I would like my boards to be more responsive and I
want named arguments as I tend to forget the argument ordering surprisingly often.
It would also be more convenient for others to operate with the board, if it responds like any
command-line program. In addition, consider the following command: `a 1 2`. What if I only want
to change the second argument and leave the first one to be default?
I cannot do that as I still have to define the first one as well (and remember the default value)
to define the second one. The named arguments would solve this too.

Introducing docopt
------------------
It was in PyCon Finland 2012 when
I heard about [docopt](http://docopt.org/), a CLI description language that
generates the required code, created by Vladimir Keleshev. I still remember
[people applauding in the YouTube video when they saw docopt in action](http://www.youtube.com/watch?v=pXhcPJK5cMc&feature=player_detailpage#t=417s) -- it really was beautiful. Yeah, Mr. Keleshev
was inhibited to give a speech in PyCon Finland 2012, so we watched his
earlier speech from YouTube. During the session I checked [the docopt's
GitHub repository](https://github.com/docopt/docopt.c) and realized that
there was a port for C. Since then I have been thinking about using docopt
with my embedded systems.

In short, using the docopt we can just write our commands as a
help message in a man page styled file and then let the docopt generate
the *docopt.c* file from this file. After then we only have to include
the generated *docopt.c* source file in our program and we are ready to use it.
Here's an example how the man page styled `.docopt` file can look like
for an imaginary board that does analog-to-digital conversion

<pre>
Usage:
  board adc measure [--average=AVG]
  board adc freerun [--timeout=SECONDS]
  board adc channel CHAN
  board adc enable | disable
  board -h | --help | --version

Options:
  -h, --help             Show this screen.
  --version              Show version.
  -a, --average AVG      Averaging [default: 1].
  -t, --timeout SECONDS  Timeout for freerun [default: 30].
</pre>

Unfortunately there was a major setback as soon as I started: I noticed
that the current C port of docopt does not support commands yet, only
options. This means that I cannot have separate commands like *measure*
to start a measurement, or *channel* to change the default channel. At the
moment I only can work with the options like *--average* or *-a* to tell
the adc unit how much the signal should be averaged.

To test this example with docopt, go to [http://try.docopt.org/](http://try.docopt.org/)
and copy-paste the upper help message into the input area there. Note that the program
name *board* is not included into the argument vector (argv).

Testing with Aery32
-------------------
Testing docopt with [Aery32](http://www.aery32.com) was quite simple
(being one of the authors the choose of platform was obvious). I copied
the *example/docopt.c* file from the docopt.c repository and placed it
to my Aery32 project folder. I then took the Serial Port class driver
example file from Aery32 framework to be my *main.cpp* and modified
it a bit, as seen below. Additionally I had to make few changes in
*docopt.c* (file) to make it play nice in an embbed environment where,
for example, one usually doesn't want to call `exit()`, but return to
`main()`. Lastly *docopt.c* source file had to be excluded in the *Makefile*
to prevent Aery32 build system from compiling it as a separate source file.

The `line_to_argv()` function was recently added to the Aery32 framework
in order to modify the read line into argument vector (argv) that can be
passed to `docopt()`. While this function was originally written the
docopt.c in mind, I've already found that it is useful in implementing
`sscanf()` free communication as well.

<pre class="prettyprint lang-c">
#include &lt;aery32/all.h&gt;
#include "board.h"

#define LED                     AVR32_PIN_PC04
#define UART0_SERIAL_PINMASK    0x3 // PA0 = RX, PA01 = TX

volatile uint8_t bufdma0[128] = {};
volatile uint8_t bufdma1[1024] = {};

using namespace aery;

periph_idma dma0 = periph_idma(0, AVR32_PDCA_PID_USART0_RX,
    bufdma0, sizeof(bufdma0));
periph_odma dma1 = periph_odma(1, AVR32_PDCA_PID_USART0_TX,
    bufdma1, sizeof(bufdma1));

serial_port pc = serial_port(usart0, dma0, dma1);

extern "C" {
    #include "docopt.c"
}

int main(void)
{
    /*
     * The default board initializer defines all pins as input and
     * sets the CPU clock speed to 66 MHz.
     */
    board::init();
    gpio_init_pin(LED, GPIO_OUTPUT|GPIO_HIGH);
    gpio_init_pins(porta, UART0_SERIAL_PINMASK, GPIO_FUNCTION_A);

    pc.set_speed(115200).enable();

    char line[32] = "";
    size_t nread = 0;

    DocoptArgs args = {};
    char *argv[8] = {};
    int argc = 0;

    for(;;) {
        pc.getline(line, &nread, '\n');
        if (nread == 0)
            continue;

        argc = line_to_argv(line, argv);
        args = docopt(argc, argv, /* help */ 1, "2.0rc2");

        pc.printf("--help == %s\n", args.help ? "true" : "false");
        pc.printf("--version == %s\n", args.version ? "true" : "false");
        pc.printf("--tcp == %s\n", args.tcp ? "true" : "false");
        pc.printf("--serial == %s\n", args.serial ? "true" : "false");
        pc.printf("--host == %s\n", args.host);
        pc.printf("--port == %s\n", args.port);
        pc.printf("--timeout == %s\n", args.timeout);
        pc.printf("--baud == %s\n\n", args.baud);
    }

    return 0;
}
</pre>

Profit?
-------
With the example code above it's possible to specify options via
terminal. For example sending a line `--baud=115200` gets parsed
and the value in `args.baud` is set accordingly. `-h` or `--help`
shows the help message. Missing arguments eg. `-x` tell that
*"-x is not recognized"*.

Now someone (I?) should complete the docopt.c and make it work with
the commands. Some work has to be carried out as well to make docopt.c friendlier for
embedded software. From an embedded point of view the return value of the `docopt()`
function could be reserved for something more useful. At the moment it returns
the populated DocoptArgs struct, which make sense when used with the common
command-line application that would just exit when done. However, in an
embedded environment we would like to reuse the *args* record, so it would make
more sense to pass the pointer to this struct and reserve the return value
for something else.

The initial structure of the DocoptArgs could also be brainstormed a bit further.
In addition, there are few bugs to fix. For example, `--foo` ends up in an infinite
loop saying *"--foo is not recognized"*, and the C++ compiler gives several warnings
of <em>"deprecated conversion from string constant to 'char*'"</em>.

Finally here's an example how the embedded main loop could look like when
everything is in place. See that I have already modified the `docopt()`
function to take a pointer to the *args* record.

<pre class="prettyprint lang-c">
int run(DocoptArgs *args)
{
	// do something with args
}

int main()
{
	// ...
	
	for(;;) {
		getline(line, &nread, '\n');
		if (nread == 0)
			continue;
		
		argc = line_to_argv(line, argv);
		docopt(argc, argv, &args, /* help */ 1, /* version */ "0.1");
		if (docopt == /* ok */)
			run(&args);
		else
			// print a message like "-x is not recognized"?
	}
	
	return 0;
}
</pre>
