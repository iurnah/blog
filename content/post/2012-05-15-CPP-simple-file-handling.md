---
title: "C++ Simple File Handling"
date: 2012-05-15
categories:
  - C++ 
toc: true;
draft: false;
---

C++ has two basic stream classes to handle files, `ifstream` and `ofstream`. To use them, include the header file `fstream`. Here is an example:

```c++
#include <fstream>
#include <iostream>
using namespace std;

int main()
{
  char str[10];

  // Creates an instance of ofstream, and opens example.txt
  ofstream a_file ("example.txt");
  // Outputs to example.txt through a_file
  a_file << "This text will now be inside of example.txt";
  // Close the file stream explicitly
  a_file.close();
  // Opens for reading the file
  ifstream b_file ("example.txt");
  // Reads one string from the file
  b_file >> str;
  // Should output 'this'
  cout<< str << "\n";
  cin.get();    // wait for a keypress
  // b_file is closed implicitly here
}
```

The default mode for opening a file with `ofstream`'s constructor is to create it if it does not exist, or delete everything in it if something does exist. If necessary, we can give a second argument that specifies how the file should be handled. They are listed below:
```
ios::app   // Append to the file
ios::ate   // Set the current position to the end
ios::trunc // Delete everything in the file
```

For example:
```c++
ofstream a_file("test.txt", ios::app);
```

This will open the file without destroying the current contents and allow you to append new data. When opening files, be very careful not to use them if the file could not be opened. This can be tested for very easily:
```c++
ifstream a_file ("example.txt");

if (!a_file.is_open()) {
  // The file could not be opened
}
else {
  // Safely use the file stream
}
```