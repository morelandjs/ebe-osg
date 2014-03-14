# EbE-OSG

Tools for running event-by-event (EbE) heavy-ion collision simulations on the Open Science Grid (OSG).



## Introduction

### What is this, and why does it exist?

I made EbE-OSG as part of a research project at Duke Physics.  It is designed to run jobs as quickly and efficiently as possible and with
minimal user intervention.

### What is it not?

A comprehensive, hold-your-hand solution.

### Who would want to use it?

Experts who want to acquire as much CPU time as possible and are willing to put in some time to configure this.  I imagine three possible
use cases:

1. Running events using the same physics models as me.
2. Using some of the same models, replacing some with your own.  I expect this is the most common.
3. Completely different models, i.e. you just want my OSG scripts.

I will address all three use cases below.


## Design

EbE-OSG provides tools for compiling and packaging software, and subsequently distributing that software on the OSG.

### The build system

Software must be compiled on the OSG submit host.  It is very unlikely that a pre-compiled binary will run successfully.

I have configured all my codes to compile with CMake.  This is a very flexible and powerful system and I strongly recommend using it,
however it is not required.  I have defined a new CMake build type "OSG" which sets the necessary compiler flags for running on the OSG.
See `lib/CMakeLists.txt` for details.

The script `lib/package` is a simple wrapper around CMake which compiles all software and packages it together.  It also provides some
convenient functions such as automatic detection of the Intel compilers.  Run `./package -h` for usage information.

### The job submission system

The OSG uses the Condor job management system.  Jobs are automatically generated and submitted to the grid by the script `submit`.  Run
`./submit` [with no arguments] for usage information.

The script `lib/remote-job-wrapper` is the Condor executable, i.e. this is the program which is actually distributed to the grid.  It
gathers host information, pulls in the necessary files, runs the simulation, and finally copies results to the destination.

The file transfer protocol is GridFTP, and is used by the remote job wrapper via the command-line utility `globus-url-copy`.  Results files
may be copied to any GridFTP location; notably, there is a GridFTP server at Duke, so results may be transferred directly from OSG machines to
Duke storage space.

### The input file system

EbE-OSG is optimized for running batches of identical jobs [identical except for the random seed].  Every time the submit script is called,
it generates a batch of identical jobs.

Every job has a core set of files that is always the same; this core set is generated by the package script.  Then, there
is a part that is different for each batch of jobs -- the input files.

The input file system is more of an idea than an actual implementation.  In my project, I use the input files to specify parameters which are
read by the executables.  However they could be anything, e.g. initial states for a model.

Input files also provide convenient labels for batches of jobs.  The submit script generates batch IDs using a timestamp and the names of
the input files.

There are a number of advantages to this system:

* Modularity.  Logical groups of parameters may be separated into different files.
* Reproducibility.  It is trivial to re-run jobs or recall parameters since they are stored in files on disk.
* Efficiency.  The main package only has to be generated once.

My input files are in the `inputs` subdirectory.

The script `lib/run-ebe` handles the input files and couples the various programs together.  The remote job wrapper calls `run-ebe`.

__Example:__  There is a file `inputs/test` in this repository that contains settings for a test event.  Suppose you want to run 10 test
events; you would execute

    ./submit 10 input/test

This generates 10 Condor jobs and submits them.  Each Condor job passes the name of the input file to `remote-job-wrapper`, i.e.

    ./remote-job-wrapper inputs/test

This is the command that is automatically executed on each OSG machine.  The remote job wrapper then passes the input file to `run-ebe`

    ./run-ebe test

The actual handling of the input file occurs entirely in `run-ebe`.  The submit script and remote job wrapper are completely agnostic to the
contents of the input files.



## Configuration

### Get an OSG account

You must be able to SSH into an OSG submit host (e.g. xsede) before doing anything.

You must have a valid grid certificate to submit jobs.  Go to https://oim.grid.iu.edu/oim/certificate to request a certificate (click
request new under user certificates on the left).  Once your certificate is approved, you will return to that site and can download the
certificate.  Look for an obvious download button, and choose the option for command-line use.  Your browser will download a file
like `user_certificate_and_key.Uxxx.p12`.  There should also be a link, how to import user certificate for command line use.  It will tell you
exactly what commands to run.  In the interest of being thorough, I reproduce those commands here:

    openssl pkcs12 -in user_certificate_and_key.Uxxx.p12 -clcerts -nokeys -out ~/.globus/usercert.pem
    openssl pkcs12 -in user_certificate_and_key.Uxxx.p12 -nocerts -out ~/.globus/userkey.pem

You must set the mode on your `userkey.pem` file to read/write only by the owner (`chmod 600 ~/.globus/userkey.pem`).

Now, verify that you can create a GridFTP proxy with `voms-proxy-init`.

### Get the source

What you do here depends on your use case.

1. If you want to use the same physics models as me, simply clone this repository on the OSG host `git clone https://github.com/jbernhard/ebe-osg.git`.
2. If you plan to add/remove physics models or make other significant changes, __fork the repository__ and then clone your fork.
3. If all you want are my OSG scripts, just download `submit` and `remote-job-wrapper`.

### Edit the submit script

Customize the variables at the top of the submit script.  Documentation is present in the file.

### Run a test job

I strongly recommend running a test job at this point.  __Please verify that you can voms-proxy-init before attempting this.__

First, compile the package:  cd into the `lib` directory and run `./package`.  This should create an archive `ebe-osg.tar.gz`.  Now, go back
up and run `./submit 10 inputs/test`.

Check the status of your jobs with `condor_q <user>`.  Jobs should finish in 5-10 minutes and the results will be placed in the location you
specified in the submit script.

If you are in use case 1, no more configuration is necessary.  You can skip the rest of this section.

### Add your projects

Anything that is configured for compiling with CMake should work well.  Edit `lib/CMakeLists.txt`, specifically the part about adding
subprojects and possibly the compiler flags.  Documentation is in the file.

### Edit the run script

This is probably the most time-consuming part.  You must create a shell script which can execute an entire event, start to finish, without user
intervention.  You might want to use my script `run-ebe` as a starting point, or start from scratch.  Note that:

* The script must be named `run-ebe`, since it is hard-coded in several other places.
* Results files must be placed in a folder `results`.  This folder is created by CMake in the packaging process.  When jobs run on the grid,
  the remote-job-wrapper copies all files in that folder; anything else will be lost.
* Anything printed to stdout/err will be recorded in the job log file.
* The run script is entirely responsible for handling the input files.  In my version, input files are parsed in INI format.
* The script is _not designed to run in place_, i.e. if you go into lib and do `./run-ebe`, it will not work at all.  It is designed to run
as part of the full OSG package.

### Compile and package

Run `./package`.  If everything works, you will have a freshly-made archive `ebe-osg.tar.gz` with your compiled binary in it.  Note that
`run-ebe` is part of the package so you must re-run `./package` whenever you change `run-ebe`.  CMake is smart and will not unnecessarily
recompile code.



## Running jobs

### Test first

I suggest running a local test job before attempting to use the grid.  Copy the package to a temporary directory, unpack it, and execute
`./run-ebe`.  Try to tune your test job to run as quickly as possible -- xsede will kill CPU-intensive processes that run for more than 20
minutes.

Once that is working, test some jobs on the grid, e.g. `./submit 10 <input files>`.

### Scale up

If everything is working, you can start submitting larger batches of jobs.

__Password automation:__ For convenience, you may place your grid password in the file `.vomspw` in ebe-osg root.  The submit script will
read the file so you don't have to type your password every time you submit a job.  Obviously, there is a significant security risk when
storing a password in plain text.  Therefore, please use a unique password and change the permissions of the file to only be readable by you
(chmod 600).

### Optimization

This system is very robust, and you should have near-perfect reliability.  I typically see >99.9% success rate.

Feel free to load up the queue with many thousands of jobs.  I routinely keep 30-50 thousand jobs in the queue, and 5-10 thousand are
running at any given time.  I like to use shell loops to submit many batches, e.g.

    for i in list_of_input_files; do
        ./submit 1000 $i
        sleep 2
    done

If possible, try to tune your jobs to take about 2-4 hours.  Shorter jobs are inefficient because it takes some time for Condor to match
your jobs and transfer the files.  Longer jobs will have other problems, such as preemption by higher-priority tasks.  I tune my job time by
running multiple sequential events in each job.

You should compress your results at the end of `run-ebe`.  Tabular numerical data can be compressed 70-80% by gzip.




## Debugging

Unfortunately there are far too many possible problems for me to address them all here.  However, I can offer some advice:

* The first thing to check is the stdout/stderr Condor logs.
* GridFTP is a common problem.  If you are getting errors from globus-url-copy, try a manual test transfer, e.g.

        globus-url-copy -v file:///full/path/to/testfile gsiftp://your.gridftp.server/full/destination/path/to/testfile

* A useful trick is `condor_ssh_to_job <job ID>`, where the job ID can be found in `condor_q` output.  This will ssh to the computer where
  your job is actually running, so you can check on the status in real time.


## Questions & comments

Please feel free to email me with any questions or feedback about how to improve EbE-OSG.
