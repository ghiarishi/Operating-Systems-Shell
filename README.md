# Penn - Shell

## Overview of Work Accomplished:

In Project 1, a C-based interactive shell has been developed. It is built on top of the Penn-Shredder, which is a basic loop that prompts, executes, and waits. The Penn-Shell includes four new capabilities, namely, standard input/output redirections, pipelines, background/foreground processing, and job control. Moreover, the shell can operate as a non-interactive shell that does not provide support for background processing and job control.

### How to run the program:
1. Start the Docker
2. In your editor, open Terminal and go to the project's directory using the cd command.
3. Enter the command to open a zsh terminal within Docker: docker exec -it cis 3800 zsh
4. Use the command 'make' to compile the code
5. Commands to run penn-shell: <br>
    * ./penn-shell : This will run the penn-shell in an interactive mode where the shell prints a prompt and waits for the user to input a command. <br>
    * ./penn-shell > filename : This runs the penn-shell in a non-interactive mode where it read commands from a script that the user has mentioned. <br>
    * .penn-shell --async :  This runs it in the asynchronous mode where it reaps all background zombies asynchronously as and when they finish executing <br>

### Work Done: 
We started out with implementing the Milestone code, which involved the creation of first, the interactive and non-interactive modes of the shell.

This essentially allows the shell to both, read commands from a file (non-interactive), or directly from the user (interactive). 

Once that was done, we moved on to adding more functionality in the form of working with standard input and output redirections. 

By default, standard input and output refer to the terminal, yet, we can have commands read input and direct output from and to certain files. This behavior is controlled through the use of (>) and (<) respectively. The files can be opened in read-only, append or overwrite modes. We can also combine these redirections to enable interesting functionality, and not use the terminal for i/p and o/p at all! 

The next major milestone was implementing pipelining. This involves the usage of pipes (specified by the character '|'), to direct the output of one process into the input of another. This allows the shell to chain together several commands and create what we shall now refer to as a Job. This exponentially increased the functionality of our shell, since now we were no longer limited to executing just one process at a time, but theoretically, thousands. 

Yet, all of this was being done only in the foreground, and the benefits of having so many, and such powerful cores in our CPU was being wasted. So the next step was to allow the existence of background processes. Step 1 here was to put each job into its own process group, which would essentially serve as a delimiter between them, allowing us to specify who to send signals to, who to take into or bring out of foreground, or take and get information from and to. To run a background process, you must type out the command, and use an '&' after it. 

Now that we have background processes, we need to wait for them properly, in order to ensure that they do not stay around as zombies. This was never a problem we faced with foreground processes, as we waited for them instantly after executing them, which never gave them a chance to be a zombie. Thus we wrote a block of code that will "poll" for background processes that have completed running by using the waitpid(2) system call. By giving the first argument as -1 and putting it in a while loop, it would wait for all the child processes that have finished running by then. Not just that, we would also print the status of each of these jobs, such as saying that they have "Stopped", are "Running" or have finished. 

With such a quick growing capability of running various processes, we require a lot more power to control them as well. This was implemented through the use of Ctrl+C, Ctrl+D, Ctrl+Z interrupt commands. Control C would kill a running foreground process, while Control Z would stop it. Neither would do anything to affect a background process. This was done through the use of SIGINT and SIGTSTP signals, which were handled through a new signal handler function. Meanwhile, Control D would just kill the entire terminal, or reprompt when there is an input present in the buffer. 

Finally, Job Control. We implemented functionality for 3 new commands, bg, fg and jobs. bg would resume in the background the stopped job identified by job_id, which defaults to that of the current job. fg would bring to the foreground the background job identified by job_id, which defaults to that of the current job. If the background job is stopped, fg resumes it before bringing it to the foreground. jobs prints to the standard error all background jobs, sorted by job_id in ascending order. 

In order to implement all this functionality, using the appropriate data structure was key. Thus, we used a queue, which was implemented using a linked list. This contained all our background and stopped jobs; and even some foreground ones in some special cases! 

## Description of code and code layout
1. Header files

2. Defined Macros

3. Created global variables

4. String Copy Function: char* strCopy(char* src, char* dest)
- Function to deep copy a string. Rather helpful inc cases where we don't want to mutate the input string. Such as with using Strtok function. 

5. Job Object: struct Job
- It stores the following variables:
    int pgid;                       // job ID
    int JobNumber;                  // Counter for current job number since first job begins from 1
    int bgFlag;                     // FG = 0 and BG = 1
    struct Job *next;               // pointer to next job
    char *commandInput;             // Input command by user (only to be used when printing updated status)
    int status;                     // tell whether its running or stopped
    int numChild;                   // Indicating the number of piped children process in PGID
    int *pids;                      // list of all pids in the job
    int *pids_finished;             // boolean array list that checks every pid is finished
    
6. Create Job Function: struct Job *createJob(int pgid, int bgFlag, int numChildren, char *input)
- Took as arguments the above stated variables, and assigned the variables of the job object to each of them. Then returned the newly created job to the requesting function. 

7. Free One Job Function: void freeOneJob(struct Job *Job)
- It frees all the memory allocated for a particular job which is passed as an argument to the function. Helps us avoid memory leaks. 

8. Free All Jobs Function: void freeAllJobs(struct Job *head)
- Same as the above function, but calling it on ALL jobs. Very helpful when exiting penn shell. 

9. Add Job to Linked List Function: struct Job *addJob(struct Job *head, struct Job *newJob)
- It adds a job to the linked list and takes as an argument the head of the linked list and the new job to be added. It then returns the head pointer of the altered linked list.

10. Remove Job Function: struct Job *removeJob(struct Job *head, int jobNum)
- Used to remove a job from the linked list. This is done very carefully, ensuring that the links present are removed if needed, and the required ones are repaired correctly. Further, freeOneJob function is also called to clear the memory being used. 
 
11. Retreive job of that Job ID function: struct Job *getJob(struct Job *head, int jobNum)
- To retrieve a job whose job number has been given as an argument to the function.

12. Get the most current job (bg or stopped): int getCurrentJob(struct Job *head)
- penn-shell has a notion of the current job. If there are stopped jobs, the current job is the most recently stopped one. Otherwise, it is the most recently created background job. This fucntion returns the job ID of the current job.

13. Change Status of Job Function: void changeStatus(struct Job *head, int jobNum, int newStatus)
- This changes the status of the job to running, stopped or finished based on the requirement by the builtin command given. 

14. Change FG/BG flag of a process: void changeFGBG(struct Job *head, int jobNum, int newFGBG)
- This function changes the bgFlag (BG to FG or vice versa) variable of the job whose job ID has been passed as an argument. 

15. Convert Status of Job to String function: char *statusToStr(int status)
- Returns the status of the job taken as integer, as string. Useful wehn we want to print the status to the terminal. 

16. Signal Handler for SIGINT, SIGTSTP, SIGCHLD: void sig_handler(int signal)
- We use this function to handle SIGINT, SIGTSTP, and SIGCHLD signals. First, we just print out a new line and the prompt. Then, we use if statements to decide what signal we are trying to handle. If it is SIGINT, a ^C has been entered into the terminal. In this case, we use the killpg() command, using SIGKILL as the flag, to kill the process groun (job) that has been specified. If it is a SIGTSTP, then we ahve encountered a ^Z. In this case, we use the kill command with SIGTSTP as the flag in order to stop the process that has been specified by the pid. 

- Finally, we have SIGCHILD. This part is active only when the async flag is high. We first block the SIGCHLD signal, as while we are already handling it, we don't want another call to this function, as that will create a race condition, which is undesirable. Then, we have a print flag, which is used to determine whether or not we want to print any statuses that we end up with after polling. In order to set it, we check if we are inside the parent process or a foreground child process. If we are in the parent, we can freely print any statuses, so the flag is low. Else it is high, as we don't want to print the statuses while we are in the midst of a child running in the foreground. Next, we perform polling, where we use waitpid(-1) to check for any state changes in any background processes. If status is stopped, when we do what is necessary, if the status is finished, then we mark it as finished, and remove the process from the linked list. Once all of the polling is done within the while loop, we can unblock the sigchild mask and return. 

17. Penn Shredder: void penn_shredder(char* buffer)
- This function has the follwing functionality:
  - It parses the input command by calling the parsed_command() method defined in parser.h
  - Checks the parsed input for errors
  - If the input command is bg, it resumes a stopped process in the BG. It checks whether a job ID has been given or not. If given, it resumes a stopped process in the BG, the job with given job ID or the most recently stopped job. If the job is already running or no such job exists, bg throws an error.
  - If the input command is fg, it brings to the foreground the backgorund job whose job ID has been given, otherwise the current job. If the background job is stopped, fg resumes it before bringing it to the foreground. If the job does not exist, fg throws an error.
  - jobs prints to the standard error all background jobs, sorted by job_id in ascending order. If there are no background jobs, it simply returns without throwing an error.
  - It then creates a child process using fork() system call. Within the child process, it checks if a file has been given for input by the user. If so, it redirects the standaer input to the file using dup2() system call.
  - If a file has been given to write the output to, it redirects standard output to that file after opening it in append or overwrite mode.
  - It then creates and redirects read/write ends of pipes to implement pipelining of commands. 
  - The execvp() system call is used to execute the command.
  - Within the parent process that created the child process(es), setpgid() system call is used to make each child process's group independent. 
  - All the file descriptors are used to close all ends of the pipes to avoid leaks.
  - A new job is created for that process and all the PIDs of the processes are stored in a list.
  - In case of a foreground process, we use tcsetpgrp() system call to give terminal control to the child.
  - We then run waitpid(-group_pid, &status, WUNTRACED) for each process in the job.
  - If we find a process that has been stopped through WIFSTOPPED(status), we change it's status to STOPPED and add it to the linked list.
  - We then give terminal control back to the parent by running tcsetpgrp().
  - During this, if there were any jobs that finished in the background, their status is printed.
  - If the input command was a bakground job, we simply add it to the queue (linked list) after executing it. 

18. Main Function: int main(int argc, char** argv)
- This is the function where it all begins. We read input here, initialize the linked lsit, call penn shredder, check for interactive or non interactive mode, etc. 
- We start out by reading the terminal arguments to main. We perform error checking, and also right here decide if we are in async mode or not. 
- We also initilize our signal handlers here in order to ensure that any and all required signals are handled incase they are encountered in the entireity of the program. We also ignore the SIGTTOU, as it does not let jobs suspend properly otherwise. 
- We first start with polling, since we frist want to see if any processes have finished (mainly for MAC). 
- Once we have polled, we check if we are in interactive or non interactive mode. If we are in non interactive, we simply read lines from the given input file, and execute them by calling pennshredder(). 
- If we are in interactive mode, then we have a lot to do: \
  - First, we write our prompt
  - Then we poll again. This is where we catch any background processes that have finished running, and reap them to ensure they don't stay as zombies, and that we can update their status and print it into the terminal. 
  - This polling also involves taking note of all the various finished processes, and we then remove them all from the linked list, also freeing the allocated memory they have used in the process. 
  - Now, we modify the buffer to make sure it is in a form that is usable to us. We check if it is of length one, or just a new line, in which case we head to the next iteration of the while loop, doing nothing. 
  - We then set the last char of the buffer to a null character to avoid memory leaks
  - Then, we implement several conditions to check for the last character of the buffer not being a new line, as that is when ctrl D has been entered. In that case, we want to kill the program. But if there is an input, we just want to reprompt. 
  - We also check right there if the input is just spaces and tabs, in which case we just reprompt. 
  - FInally, we call penn shredder, whos functionality we have already gone over. 
  - We then set teh current job to the head, as we want to now be able to iterate through the linked list, since we have one or more jobs. 
  - Before exiting main (the entire program), we run freeAllJobs, free(buggerSig), and a few other free()s to ensure there are no memory leaks. This is also done when the program exits due to failure or an error anywhere. 

## Additional Functionality: Reap background zombies asynchronously

The SIGCHLD signal is ignored by default. In the regular implementation, penn-shell polls the background jobs and reaps zombies if any. This may leave some zombies around indefinitely. If we run a process in the background it becomes a zombie when it completes execution. It will stay in the zombie state until penn-shell gets around to reaping it when polling background processes the next time.
 
Asynchronous SIGCHLD handling has been implemented to reap zombies in case of '--async' command-line option passed to penn-shell. 
    
To handle signaling child processes on demand, a SIGCHLD handler (sig_handler(int signal)) has been registered. Its functionality has been described above. <br>

SIGCHLD is generated whenever a child changes state (e.g., running to stopped, running to exited, etc.), and this will invoke the signal handler.

In the sighandler, we first block the SIGCHLD signal using sigempryset, sigset, sigaddset and sigprocmask. This is to avoid there being a race condition if there is a SIGCHLD thrown while we are already handling one. This is unblocked at the end. Then, within this function, we have implemented functionality that you can find within the sighandler function writeup in this file, but in short, we check if we have to or dont have to print the status of processes, perform polling, use a buffersig malloc incase we need to store these new statuses, and then remove any finished processes from the linked list. 

To make MAC work, we also had to perform polling in a few other locations as zombie background processes could finish at a variety of times, where we would need to reap them and store/print their status. 

We would print the status of jobs ONLY after a foreground process has finished execution, since we would not want to interrupt them. 

