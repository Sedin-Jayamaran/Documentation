# 10 Essential Shell Scripts (80-20 Rule)

This reference covers the 10 core shell scripts that represent **80% of practical shell scripting for beginners**. Each script is explained with its code, purpose, and a detailed beginner-friendly guide to help you master the fundamentals quickly using the 80-20 principle (focus on the 20% that gives 80% of the results).

## 1. Print a Message to Terminal

```bash
#!/bin/bash
# Prints "Hello World" to the terminal
echo "Hello World"
```
**Explanation**: This is your first script! The `echo` command displays simple text. The first line (`#!/bin/bash`) is always at the start — called the "shebang" — it tells the computer to use the bash interpreter to run your script.

## 2. Assign and Print Variables

```bash
#!/bin/bash
name="Jayesh"
age=21
echo $name $age
```
**Explanation**: You assign values using `=` with **no spaces**. The `$name` and `$age` print the values of the variables. Variables are fundamental to scripts.

## 3. Take User Input and Greet Them

```bash
#!/bin/bash
echo "What's your name?"
read name
echo "Hello, $name! Nice to meet you."
```
**Explanation**: The `read` command takes input from the user and stores it in a variable. The script then uses that input to display a personalized message.

## 4. Check If File Exists

```bash
#!/bin/bash
file="example.txt"
if [ -e "$file" ]; then
  echo "File exists: $file"
else
  echo "File not found: $file"
fi
```
**Explanation**: The `if [ -e "$file" ]` checks if a file exists. The `-e` flag is used for this. This form is essential for automating checks before running actions on files.

## 5. Use Command-Line Arguments

```bash
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
```
**Explanation**: `$0` gives the script name, `$1` and `$2` give arguments entered by the user in the command line. This makes scripts flexible and reusable with different inputs.

## 6. Loop Over a List (for Loop with Array)

```bash
#!/bin/bash
fruits=("apple" "banana" "cherry" "date")
for fruit in "${fruits[@]}"; do
  echo "Current fruit: $fruit"
done
```
**Explanation**: Arrays store multiple values. The `for` loop goes through each value in the array and prints it. This is the foundation of batch processing and data automation in scripts.

## 7. Sum Numbers from 1 to N Using a Loop

```bash
#!/bin/bash
echo "Enter a number (N):"
read N
sum=0
for (( i=1; i<=$N; i++ )); do
  sum=$((sum + i))
done
echo "Sum of integers from 1 to $N is: $sum"
```
**Explanation**: `for (( i=1; i<=$N; i++ ))` is numeric looping — useful for repeated calculations. `$((sum + i))` is arithmetic expansion.

## 8. Read Each Line of a File

```bash
#!/bin/bash
file="sample.txt"
if [ -e "$file" ]; then
  while IFS= read -r line; do
    echo "Line read: $line"
  done < "$file"
else
  echo "File not found: $file"
fi
```
**Explanation**: The `while read` loop processes each line in a file, making it great for log processing or data extraction. `IFS=` preserves line format.

## 9. Basic Conditional Statement (if-elif-else)

```bash
#!/bin/bash
number=5
if [ $number -gt 10 ]; then
  echo "$number is greater than 10"
elif [ $number -eq 10 ]; then
  echo "$number is equal to 10"
else
  echo "$number is less than 10"
fi
```
**Explanation**: `if`, `elif` (else if), and `else` provide decision-making capability, controlling which code runs based on conditions.

## 10. Find the Greatest Element in an Array

```bash
#!/bin/bash
array=(3 56 24 89 67)
max=${array[0]}
for num in "${array[@]}"; do
  if ((num > max)); then
    max=$num
  fi
done
echo "The maximum element in the array is: $max"
```
**Explanation**: This script finds the largest number in a list using a loop and comparison. Arrays + loops + if statements are the backbone of practical scripting!

---

Each example above covers the essential concepts: echo, variables, input, file handling, arguments, loops, conditionals, and arrays. **Understanding these 10 scripts gives you the core 80% of shell scripting knowledge applicable in real jobs and interviews.**

For more examples and explanations, revisit this markdown file anytime as your shell scripting cheat sheet!

---