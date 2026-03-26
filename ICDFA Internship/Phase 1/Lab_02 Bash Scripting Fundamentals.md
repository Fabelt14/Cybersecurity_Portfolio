# Bash Scripting Fundamentals

## Overview

This lab focused on writing Bash scripts to automate tasks and build programming logic in Linux. The goal was to move beyond running individual commands to creating reusable scripts that handle user input, make decisions, loop through operations, and monitor system resources.

## Objectives

- Capture and process user input in scripts
- Implement conditional logic for decision-making
- Use loops for repetitive tasks
- Create functions for code reusability
- Monitor system resources programmatically
- Build practical automation tools for security work

## Lab Environment

- **OS**: Kali Linux
- **Shell**: Bash (/bin/bash)

## Tools Used

- nano - Script editing
- bash - Script execution
- chmod - Making scripts executable
- date - Time/date retrieval
- free - Memory statistics
- ping - Network connectivity testing
- awk - Text processing for data extraction
- grep - Pattern matching

## Methodology

### Exercise 1: User Input Script

#### Step 1: Understanding User Input with `read`
I needed to create a script that asks for user preferences (favorite color and food) and displays them back.

**Why this matters in security:**
Many security tools require user input - target IP addresses, port ranges, scan types. Understanding how to capture and validate input is fundamental.

**Script breakdown:**
```bash
#!/bin/bash
echo "Hello User!"
read -p "What is your favorite colour? " colour
read -p "What is your favorite food? " food
echo "Hello User, your favorite colour is $colour and your favorite food is $food"
```

**How `read` works:**
- `-p` flag displays a prompt and waits for input on the same line
- Input is stored in the variable name that follows (colour, food)
- Variables are accessed with `$variable_name`

![User Input Script Execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.1%20User%20Input%20Script%20Execution.jpg)

**Testing:**
- Input: Purple, Beans
- Output: "Hello User, your favorite colour is Purple and your favorite food is Beans"

### Exercise 2: Password Checker with Conditionals

#### Step 1: Building Authentication Logic
I created a simple password checker that compares user input against a hardcoded password.

**Why this is relevant:**
This demonstrates basic authentication logic. Real password systems hash passwords instead of plaintext comparison, but the conditional flow is the same.

**Script breakdown:**
```bash
#!/bin/bash
read -p "Hello User! Kindly input your password: " Password

if [ $Password == secret123 ]; then
    sleep 2s
    echo "Password is Correct, Successfully login....."
else
    sleep 2s
    echo "Password is Incorrect! Try Again....."
fi
```

**Conditional syntax explained:**
- `if [ condition ]; then` - starts conditional block
- `==` - equality comparison (also could use `-eq` for numbers)
- `sleep 2s` - pauses execution for 2 seconds (simulates processing)
- `else` - runs if condition is false
- `fi` - closes the if block

**Security flaw in this script:**
The password is visible in plaintext when typed. Real implementations use `read -s` (silent mode) to hide input.

![Password Checker Script Execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.2%20Password%20Checker%20Script%20Execution.jpg)

**Testing:**
- Correct password (secret123): "Password is Correct, Successfully login....."
- Wrong password (anything else): "Password is Incorrect! Try Again....."

### Exercise 3: Countdown Script with While Loop

#### Step 1: Implementing Loop Logic
I built a countdown timer that counts from 10 to 1, then prints "Blast Off!"

**Use case in security:**
Countdown timers are used in:
- Timed scans (run nmap for 60 seconds then stop)
- Rate limiting (wait X seconds between connection attempts)
- Timeout mechanisms (abort if no response after 30 seconds)

**Script breakdown:**
```bash
#!/bin/bash
counter=10

while [[ $counter -gt 0 ]]
do
    echo $counter
    sleep 1s
    ((counter--))
done

sleep 2s
echo "Blast Off!"
```

**Loop mechanics:**
- `counter=10` - initialize variable
- `while [[ $counter -gt 0 ]]` - loop while counter is greater than 0
- `do ... done` - defines loop body
- `((counter--))` - decrements counter by 1 each iteration
- `-gt` - "greater than" comparison

**Why double brackets `[[ ]]`:**
Double brackets support more advanced comparisons and are safer than single brackets for string operations.

![Countdown Script Execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.3%20Countdown%20Script%20Execution.jpg)

**Output:**
Displays 10, 9, 8, 7, 6, 5, 4, 3, 2, 1 with 1-second delays, then "Blast Off!" after 2 seconds.

### Exercise 4: Sum Function with Parameters

#### Step 1: Creating Reusable Functions
I wrote a function that takes two numbers and returns their sum.

**Why functions matter:**
In security scripting, you might need to:
- Calculate subnet ranges
- Convert between number systems
- Parse and process data multiple times

Functions let you write the logic once and call it anywhere.

**Script breakdown:**
```bash
#!/bin/bash
read -p "Enter Your First Number: " num1
read -p "Enter Your Second Number: " num2

sum() {
    calculate=$(( $1 + $2 ))
    echo $calculate
}

result=$(sum "$num1" "$num2")
echo "The sum of the two numbers is: $result"
```

**Function mechanics:**
- `sum() { ... }` - defines function named "sum"
- `$1` and `$2` - function parameters (first and second arguments)
- `$(( arithmetic ))` - performs integer math
- `result=$(sum "$num1" "$num2")` - calls function and captures output
- Quotation marks around variables prevent word splitting

**Why capture in a variable:**
Instead of echoing directly in the function, capturing the result lets you use it in further calculations or conditionals.

![Sum Function Script Execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.4%20Sum%20Function%20Script%20Execution.jpg)

**Testing:**
- Input: 5, 9
- Output: "The sum of the two numbers is: 14"

### Exercise 5: Time-Based Greeting Script

#### Step 1: Getting System Time
I created a script that greets users differently based on the time of day.

**Security application:**
Log entries need timestamps. Scripts might need to run different tasks at different times (run scans during off-hours, generate reports at end of day).

**Script breakdown:**
```bash
#!/bin/bash
hour=$(date +%H)
display_time=$(date +%H:%M)

if [[ $hour -lt 12 ]] ; then
    echo "Good Morning! User...."
    sleep 1s
    echo "The 🕔time is $display_time"
elif [[ $hour -lt 18 ]] ; then
    echo "Good Afternoon! User...."
    sleep 1s
    echo "The 🕔time is $display_time"
else
    echo "Good Evening! User...."
    sleep 1s
    echo "The 🕔time is $display_time"
fi
```

**How `date` command works:**
- `date +%H` - extracts hour in 24-hour format (00-23)
- `date +%H:%M` - extracts hour and minute (e.g., 14:35)
- `$(command)` - command substitution, runs command and stores output

**Logic flow:**
- If hour < 12: Morning (midnight to 11:59 AM)
- Else if hour < 18: Afternoon (noon to 5:59 PM)
- Else: Evening (6:00 PM to 11:59 PM)

**Why `elif` instead of multiple `if`:**
`elif` creates a chain where only one block executes. Multiple `if` statements would check all conditions even after finding a match.

![Greeting Script Execution](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.5%20Greeting%20Script%20Execution.jpg)

**Testing at 10:24:**
Output: "Good Morning! User...." followed by "The 🕔time is 10:24"

### Exercise 6: Network Connectivity Monitor

#### Step 1: Building Automated Monitoring
I created a script that continuously checks internet connectivity and logs the status.

**Real-world security use:**
Network monitoring scripts alert you to:
- Connectivity drops during penetration tests
- Target systems going offline
- Network changes that affect tool access

**Script breakdown:**
```bash
#!/bin/bash
while true
do
    ping -c 3 google.com > /dev/null 2>&1
    status=$?
    
    if [[ $status -eq 0 ]] ; then
        echo "$(date) + $status: Connection is Active!" >> Network.log
    else
        echo "$(date) + $status: Connection is Down!" >> Network.log
    fi
    
    sleep 5
done
```

**Breaking down the ping command:**
- `ping -c 3 google.com` - sends 3 ping packets to google.com
- `> /dev/null` - redirects standard output to nowhere (suppresses normal output)
- `2>&1` - redirects error output to same place as standard output
- `$?` - special variable containing exit status of last command (0 = success, non-zero = failure)

**Why redirect to /dev/null:**
We only care if ping succeeded or failed, not the actual ping statistics. Suppressing output keeps logs clean.

**Infinite loop logic:**
- `while true` creates an endless loop
- Script runs until manually stopped (Ctrl+C)
- `sleep 5` waits 5 seconds between checks

**Log format:**
Each entry includes timestamp, exit status code, and connection state.

![Network Monitor Log Output](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.6%20Network%20Monitor%20Log%20Output.jpg)

**Sample log entries:**
```
Fri Dec 19 09:10:14 AM WAT 2025 + 0: Connection is Active!
Fri Dec 19 09:10:19 AM WAT 2025 + 0: Connection is Active!
Fri Dec 19 09:12:08 AM WAT 2025 + 0: Connection is Down!
```

The `+ 0` indicates successful ping (exit status 0). If connection failed, status would be non-zero.

### Exercise 7: System Health Monitor (Memory)

#### Step 1: Extracting Memory Statistics
I built a script that checks available memory and alerts if running low.

**Why this matters in security:**
Memory-intensive tools like password crackers, hash analyzers, or packet captures can crash if memory runs out. Monitoring prevents unexpected failures during critical operations.

**Script breakdown:**
```bash
#!/bin/bash
available=$(free -m | grep Mem | awk '{print $4}')
used=$(free -m | grep Mem | awk '{print $3}')
total=$(free -m | grep Mem | awk '{print $2}')

if [[ $available -lt 1024 ]] ; then
    echo "Checking Memory........."
    sleep 2s
    echo "Still Checking Memory........."
    sleep 2s
    echo "Almost done.........."
    sleep 3s
    echo "====================================================="
    echo " CRITICAL: Memory is running low! Only ${available}MB left"
    echo "======================================================"
else
    echo "Checking Memory........."
    sleep 2s
    echo "Still Checking Memory........."
    sleep 2s
    echo "Almost done.........."
    sleep 3s
    echo "========================================================"
    echo " GOOD: Memory Health is OK! You've used ${used}MB of ${total}MB"
    echo "========================================================"
fi
```

**Command pipeline explained:**
```bash
free -m | grep Mem | awk '{print $4}'
```

1. `free -m` - displays memory in megabytes
2. `| grep Mem` - filters to the "Mem:" line only
3. `| awk '{print $4}'` - extracts 4th column (available memory)

**Sample `free -m` output:**
```
              total        used        free      shared  buff/cache   available
Mem:           3572         932        2104          37         535        2310
```

The script extracts:
- Column 2 (total): 3572
- Column 3 (used): 932
- Column 4 (free): 2104

**Threshold logic:**
If available memory < 1024 MB (1 GB), trigger critical alert.

**Why the sleep commands:**
Creates a "checking..." effect that simulates processing time. In production, you'd remove these and run checks instantly.

![Memory Health Monitor Output](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/02.7%20Memory%20Health%20Monitor%20Output.jpg)

**Testing results:**
- System with 2310 MB available: "GOOD: Memory Health is OK! You've used 932MB of 3572MB"
- If available dropped below 1024 MB: "CRITICAL: Memory is running low! Only [X]MB left"

## Findings

**User Input Handling:**
- The `read` command captures input into variables
- `-p` flag allows inline prompts instead of separate echo statements
- Input validation (checking if input is empty, numeric, etc.) was not implemented but would be needed in production scripts

**Conditional Logic:**
- Single brackets `[ ]` work for basic comparisons
- Double brackets `[[ ]]` handle strings better and support pattern matching
- `==` for string equality, `-eq` for numeric equality
- `elif` creates decision chains where only one branch executes

**Loop Mechanisms:**
- While loops check condition before each iteration
- `while true` creates infinite loops (useful for monitoring)
- Loop variables need initialization before the loop
- `((variable--))` syntax handles arithmetic operations

**Function Design:**
- Functions use `$1, $2, $3...` for parameters
- Functions return values via echo (captured with command substitution)
- Quoting variables in function calls prevents word splitting issues

**System Information Extraction:**
- `date` command provides timestamps in various formats
- `free -m` shows memory statistics in megabytes
- Piping commands (free | grep | awk) extracts specific data points
- Exit status `$?` captures success/failure of previous command

**File Operations:**
- `>> filename` appends output to file (creates if doesn't exist)
- `> /dev/null` discards output
- `2>&1` redirects errors to same destination as standard output

## Challenges Faced

**Arithmetic in Bash:**
Initially tried `result = $num1 + $num2` which failed because Bash treats this as string concatenation. Learned that arithmetic requires special syntax: `$(( expression ))` or `(( variable++ ))`.

**Variable quoting:**
When passing variables to functions, unquoted variables can break if they contain spaces. For example, `sum $num1 $num2` fails if num1="5 6". Using quotes `sum "$num1" "$num2"` prevents this.

**Exit status confusion:**
The `$?` variable initially seemed backwards - success is 0, failure is non-zero. This is Unix convention: 0 means "no errors", any other number indicates a specific error code.

**Infinite loop stopping:**
The network monitor ran indefinitely. Learned that Ctrl+C sends SIGINT (interrupt signal) to stop running scripts. For production monitoring, would need proper daemon management.

**Awk column extraction:**
The `free -m` output format wasn't immediately clear. Running `free -m | grep Mem` first helped visualize which column number corresponded to which memory type before writing the awk extraction.

## Key Takeaways

- **Scripts automate repetitive tasks:** Instead of manually checking network connectivity every 5 seconds, one script does it forever while I work on other things
- **Variables store and reuse data:** Capturing user input, command output, or calculations in variables makes scripts flexible and reusable
- **Conditionals enable decision-making:** Scripts can respond differently to different inputs or system states (password correct/incorrect, memory high/low)
- **Loops handle repetition:** Whether counting down from 10 or monitoring network status indefinitely, loops eliminate manual repetition
- **Functions promote code reuse:** Write once (sum function), call multiple times with different inputs
- **Exit status indicates success/failure:** The `$?` variable lets scripts react to whether previous commands succeeded or failed
- **Command piping extracts specific data:** Combining free, grep, and awk in a pipeline extracts exactly the memory statistic needed without manual parsing
- **Security tools rely on these fundamentals:** Password checkers, network monitors, and system health checks demonstrate the building blocks of larger security automation

## Disclaimer

This lab was performed in a controlled Kali Linux environment for educational purposes as part of the ICDFA Linux Operating Systems Fundamentals course. All activities were conducted on a local system with proper authorization.
```
