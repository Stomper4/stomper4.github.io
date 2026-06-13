---
layout: page
title: AL1C3CTF
description: Creating challenges for a SAIT CTF
img: assets/img/projects/AliceInPwnderland_mainlogo.png
importance: 1
category: CTF
related_publications: false
---

I recently had the opportunity to volunteer to help out with a Capture the Flag competition hosted by SAIT (Southern Alberta Institute of Technology). The CTF was called Alice in Pwnderland.

As part of the technical team, I created 4 reverse engineering challenges for the contestants to solve. My goal was to make my challenges educational. I based the challenges on skills learned in class in the ISS program to make them more accessible to contestants who were new to CTFs.

The full challenges and walkthroughs can be found here: <a href=https://github.com/Stomper4/AL1C3CTF>AL1C3CTF Challenges and Write-up</a>. The rest of this article is a summary of how I structured the challenges and what I hoped the contestants could learn.
<br><br>

## Main Concepts

The intended solves for my challenges would take contestants through a variety of dynamic analysis reverse engineering skills, especially focusing on changing the control logic of a program to unlock additional functionality. I struggled to decide whether to add complexity to potential alternate solve paths. In the end I did not, if a contestant was good at static analysis and wanted to do it that way, I didn't want to prevent it. Hopefully they could look through my walkthroughs and learn new techniques for dynamic analysis.
<br><br>

### Challenge 1: Manipulating Code

<a href="https://github.com/Stomper4/AL1C3CTF/blob/main/Mad%20Hatters%20Message/Walkthrough1/RE_chal1-walkthrough.md">Mad Hatter's Message Write-up</a>

This challenge provided contestants with obfuscated source code written in C. The source code was for a program which took 5 integer arguments and if they were correct, it printed the flag. If contestants were taking the ISS course at SAIT, this challenge required skills learned in the first semester programming class.

The difficulty I had was to hide the flag in such a fashion that it wasn't just an encryption challenge. The contestant couldn't find the flag variable in the source code and attack the obfuscation of just that variable. To accomplish this, I wrote the code to derive the flag from the integer arguments entered by the user. The arguments were rotated for obfuscation, then tested against specific keys to check if they were correct. Then I created a character pointer with the same address as the integer pointer. This allowed me to work with integers in the source code, but print the flag as a string of letters.

Working with the source code allowed users to re-write the logic flow of the program. My write up demonstrates how a few small changes to the code cause the program to print the flag every time it is executed, without the correct arguments.
<br><br>

### Challenge 2: Function Pointers

<a href="https://github.com/Stomper4/AL1C3CTF/blob/main/Knave%20of%20Hearts'%20Payload/Walkthrough2/RE_chal2-walkthrough.md">Knave of Hearts Payload Write-up</a>

This challenge provided contestants with shellcode that they could execute on a kali machine to print the flag. It was very easy to make (I used msfvenom) and the goal was to have contestants explore how to execute shellcode from memory. Contestants could load the code in any way they wanted, in my write up I had a basic C program that used a function pointer to load shellcode into memory, and then execute it.

If contestants were taking the ISS program, They would have done this multiple times in the 3rd semester exploitation class.
<br><br>

### Challenge 3: Live Debugging

<a href="https://github.com/Stomper4/AL1C3CTF/blob/main/Caterpillar's%20Binary/Walkthrough3/RE_chal3-walkthrough.md">Caterpillar's Binary Write-up</a>

This is the first challenge that supplied the contestants with a binary file rather than an ascii file. It was a very simple program that asks the user for a password, and if they enter the password it prints the flag. In the binary I stored the password as a hash, and the flag was encrypted, so printing ascii strings from the binary would not immediately show the answer.

The intended solve was to use GDB (debugger) to explore both program flow options after the program checked the password. If the contestants were taking the ISS program, they would have learned how to do this in the second semester computer architecture class.
<br><br>

### Challenge 4: Altering a File

<a href="https://github.com/Stomper4/AL1C3CTF/blob/main/The%20Mirror%20is%20Broken/Walkthrough4/RE_chal4-walkthrough.md">The Mirror is Broken Write-up</a>

This challenge was a little more CTF style random chaos. If there were experienced players that had breezed through the last 3 challenges quick, this is where I wanted to hang them up. The twist was that I reversed the entire binary, the last byte was first, etc. You could discover this by running strings, seeing they were backwards, then looking for backwards file headers to determine the file type and best way to fix. Once the file was reversed, it played a similar game as Challenge 3. The main change was a parent process that killed all execution after 5 seconds. Contestants who were debugging, no longer had time to explore all the potential program flows.

My write up shows a similar attack method as I used in challenge 3, but using a different tool. I still focus on the jump logic after checking the password. Instead of using GDB debugger to change the flag in real time, I used Ghidra to overwrite the jump instruction and change the flow.

<br>
*Thanks for Reading*