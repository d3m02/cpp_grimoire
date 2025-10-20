
## C++ String Interface 
#### Note based on cppreference 
In cppreference description provided for `std::basic_string`. In fact, it's template which depends on template argument create "different" string:

| Type                          | Definition                         |
| ----------------------------- | ---------------------------------- |
| `std::string`                 | `std::basic_string<char>`          |
| `std::wstring`                | `std::basic_string<wchar_t>`       |
| `std::u8string` (C++20)       | `std::basic_string<char8_t>`       |
| `std::u16string` (C++11)      | `std::basic_string<char16_t>`      |
| `std::u32string` (C++11)      | `std::basic_string<char32_t>`      |
| `std::pmr::string` (C++17)    | `std::pmr::basic_string<char>`     |
| `std::pmr::wstring` (C++17)   | `std::pmr::basic_string<wchar_t>`  |
| `std::pmr::u8string` (C++20)  | `std::pmr::basic_string<char8_t>`  |
| `std::pmr::u16string` (C++17) | `std::pmr::basic_string<char16_t>` |
| `std::pmr::u32string` (C++17) | `std::pmr::basic_string<char32_t>` |
|                               |                                    |
`::pmr` - polymorphic memory resource.

Beside that exists `std::basic_string_view` (with same tempalte-implementation idea) which contain some pretty identical member "r/o" functions (`find`, `contains`, `starts_with` etc)

In cppreference also used `ChartT`, `size_type` etc since examples taken from template functions and real type depends on string's template argument (`char`/`wchar_t`) etc.

### Searching Strings
* `find(const basic_string& str, size_type pos = 0)`, `find(const CharT* s, size_type pos = 0)`, `find(CharT ch, size_type pos = 0)` (signature mean that we can use either string of single char for searching) return index of first match after `pos`. If character not found `std::string::npos` will be returned
* There is `rfind()` member function that finds the last occurrence of its argument (function signature same);
* `find_first_of(const basic_string& str, size_type pos = 0)`, `find_first_of(const CharT* s, size_type pos = 0)` - find first occurrence of any character from the argument string
* Similarly for `find_last_of` - find last occurrence 
* `find_first_not_of`/`find_last_not_of` search for the first (last) occurrence of any character not in the argument

#### from cppreference 
* `find(const CharT* s, size_type pos, size_type count)` (and similar) - here looks like "search substring from s" (similarly to append), but in reality it's "find first `count` symbols from argument C-string `s` in original string starting from `pos`) 🤦
* `find_first_of(CharT ch, size_type pos = 0)` equivalent to `find(CharT ch, size_type pos = 0)`; same thing with `find_last_of(CharT ch, size_type pos = npos)` == `rfind(CharT ch, size_type pos = npos)`
* `find_first_not_of(CharT ch, size_type pos = 0)` in general equivalent to `find_last_not_of(CharT ch, size_type pos = npos)` since with only 1 character we can check that it's not exists in string. It's also can be used to check that string contain only one exact character (although `find_last_not_of` search from end to begin, to check that symbol not found we compare with `std::string::npos` and not 0, since 0 is index of first character)  
```c++
std::string str{"aaaaa"};
if (str.find_first_not_of('a') == std::string::npos)
    std::cout << "all of them a!\n";
if (str.find_last_not_of('a') == std::string::npos)
    std::cout << "all of them a!\n";
```


### Adding Elements to String
* `append(const basic_string& str, size_type pos, size_type count)` - equivalent to `append(str.substr(pos, count))`, add substring to current string
* `insert(size_type index, const basic_string& str)`- insert string before `original_str[index]`
```c++
std::string str{"for"};
str.insert(2, "lde"s); // string is now "folder"
```
* `insert(size_type index, const basic_string& str, size_type s_index, size_type count = npos)` - insert substring `str[s_index, count)` 
* `insert(size_type index, size_type count, CharT ch)` - insert a char multiple times
* We can also use iterators, but remember that iterators can be invalid after modifying string (if allocation required after insert)
```c++
std::string str{"word"};
str.insert(std::prev(str.end()), 'l'); //str is now "world"

std::string str2{"ski"};
str2.insert(str2.end(), 2, 'l'); // str2 is now "skill"
```

### Removing Elements from Strings
* `erase()` takes two arguments - index of the first element to be erased and number elements to erase. For position of first element we can `find` (just remember to check that character exist) 
```c++
std::string str{"Hello"};
if (auto pos = str.find('e'); pos != std::string::npos)
	str.erase(pos, 2); // string now "Hlo"
```
* We can also use iterators. If use single iterator - it will erase the corresponding element, with iterator range - it will erase all elements in the half-closed range (`[begin, end)`)
```c++
str.erase(std::next(str.begin()), std::prev(str.end())); // Erase all character except first and last
```
* `replace()` will remove some of the characters from a string and replace them with other characters. The first argument is index of first character to be remove, the second argument is the number of character to be removed and the remaining argument give the characters that will be inserted 
```c++
std::string str{"Say Hello"};
str.replace(str.find('H'), 5 "Goodbye");
```
* Similarly to `erase()` we can use iterators. 
```c++
str.replace(str.begin(), str.begin()+3, "Wave"); // str now "Wave Goodbye"
```
* `assign()` can replace all the characters from a string with other characters

### Converting between Strings and Number
* `std::to_string` convert any numeric number to string.
* ` stoi (const std::string& str, std::size_t* pos = nullptr, int base = 10);` takes `std::string` argument and return it as an int. Leading whitespaces are ignored, but any non-number character terminate function.
```c++
std::string str{" 314 159"};
int val = std::stoi(str); // val = 314
```
* An optional second argument gives the number of character which were processed. If there is no errors, this is equal to the string's size, otherwise we get index of the first non-numeric character. _If the conversion fails completely, it throws an exception_
```c++
std::string str {" 123 "};
try {
    size_t n_processed;
    int val = std::stoi(str, &n_processed);
    if (n_processed < str.size())
        std::cout << "Processed only " << n_processed;
}
catch (std::invalid_argument e) {
    std::cout << "Not a number\n";
}
catch (std::out_of_range e){
    std::cout << "Can't fit into type";
}
```
* By default string is assumed to be decimal by default. An optional third argument gives the base (can be 2-36, 0 - the numeric base is auto-detected)
```c++
std::cout << std::stoi("2a", nullptr, 16); // will display 42
```
* For `double` can be used `std::stod()`, it's pretty same but don't have option to use different bases

#### cppreference 
Beside `std::stoi` and `std::stod` also exists 

| Returned type        | Function        |
| -------------------- | --------------- |
| `long`               | `std::stol()`   |
| `long long`          | `std::stoll()`  |
| `unsigned long`      | `std::stoul()`  |
| `unsigned long long` | `std::stoull()` |
| `float`              | `std::stof()`   |
| `long double`        | `std::stold()`  |
Strange that conversion to `unsigned int` not exists, one of theory is that it's due to size equality with `unsigned long` in most platforms (plus in some implementations range changed against `unsigned`)

### Miscellaneous String Operations
* `std::string` and `std::vector` have a `data()` member function, which return a pointer to the container's internal c-style memory buffer. For `std::string` this is null-terminated and equal to `c_str()`. It's useful for working with APIs written in C 
```c++
void print (int* array, size_t size);

std::vector<int> number {1, 2, 3, 4};
print(numbers.data(), numbers.size());
```
* `std::string` has `swap()`, which also implemented to support global `swap(T left, T right)` function.

### Character Functions
* C++ STL has a number of character functions which are inherited from C, these are defined in `<cctype>` and operate on `char`. 
	* `std::isdigit(c)` - returns `true` if `c` is a digit
	* `std::islower(c)` - returns `true` if `c` in lower case
	* `std::isupper(c)` - return `true` if `c` is upper case
	* `std::isspace(c)` - return `true` if `c` is a whitespace character
	* `std::ispunct(c)` - return `true` if `c` is a punctuation character
* `std::tolower()` and `std::toupper()` to convert to character case equivalent.

#### from cppreference
All of them take as argument `int` since function checks character ASCII code. Also in standard exist
* `std::isalnum()` - checks if a character is alphanumeric (numbers, lower and upper case letters)
* `std::isalpha()` - checks if a character is alphabetic (lower and upper case letters)
* `std::isxdigit()` - checks if a character is a hexadecimal character
* `std::iscntrl()` - checks if a character is a control character
* `std::isgraph()` - checks if a character is a graphical character (digits, letters, punctuation)
* `std::isblank()` - checks if a character is a blank character (`0x20` and `0x09`)
* `std::isprint()` - checks if a character is a printing character (isgraph + space)

## Files and Streams
#### Note based on cppreference
Similarly to strings, streams in cppreference described as templates `basic_{something}stream` since stream template can be specialized with template parameter of type `char` and `wchar_t`: `std::ifstream == std::basic_ifstream<char>`, `std::wifsteam == std::basic_ifstream<wchar_t>`


### Files and Streams 
Files interaction are represented by `fstream` objects, accessed as sequence of bytes in order of unknown length with no structure (without any understanding of file format). 
`fsteram` operations includes 
* `open` - connect the `fstream` object to the file, the file becomes available for used by the program (locked?)
* `read` - data is copied from the file into the program's memory
* `write` - data is copied from the program's memory to the file
* `close` - disconnect the `fstream` object from the file, the file is no longer available for use by the program. 
For each if these operations, the `fstream` object will call a function in the operating system API. The program will stop and wait while the operation is performed. OS can also restrict amount of opened files by the program, this is also one of reasons why the file should be closed after use.
During reading/writing data to file data may be stored in temporary memory buffer. It's done to reduce downtime during interacting with physical device. 

### File Streams
In C++ commonly used two types of `fstream`: 
* `ofstream` - file stream for writing
* `ifstream` - file stream for reading
As there are many different files on a computer, we need to associate a file stream object with the file we are using. The easiest way to open a file is to pass its name to the `ifstream` constructor. This represents a "communication channel" which is used to receive data from the file. Constructor can failed to open file, thus before accessing file it worth it to check is file opened or not.  
```c++
#include <fstream>
std::ifstream ifile("text.txt");
if (ifile)
	std::cout << "File opened\n";'
```
>The way how `if (ifile)` work is due to implementation of [`operator bool()`](https://en.cppreference.com/w/cpp/io/basic_ios/operator_bool) in base class `std::basic_ios`

We can use `ifile` the same way as `std::cin`. This will read one word at a time and remove all whitespace from the input. But it's not ideal way to work with file stream since it's maybe be not suitable for data structures and can be difficult for error-handling
```c++
std::string text;
while (ifile >> text)
	std::cout << text << ", ";
```
Often it's easier to read a complete line of input from the file with `getline()` functions. It can be either `std::getline(input_stream, string)` or stream member function, function also support optional argument for delimiters. Into input string will be copied whole line except newline character. When end of file will be reached, `getline()` will return `false`, meaning that function can be used inside loops
```c++
while (std::getline(ifilem, text))
	std::cout << text << '\n';
```

For writing we create a stream object `ofstream`, can write same way as `std::cout`. In fact there no opposite to `getline`.

When `fstream`'s destructor is called, the file is automatically closed, any unsaved data will be written to the file. If and `fstream` object goes out of scope after we have finished with it, we don't need to explicitly call `close()`, however it's good practice to do so.

### Streams and Buffering 
C++ streams use "buffering" to minimize calls to the operating system. During write operations, data is temporary held in a memory buffer, the size of this buffer is chosen to match the maximum amount of data that the operating system will accept. When the buffer is full, the steam will remove the data from the buffer and send it to the operating system. This is known as "flushing" the output buffer. 
For `ostream` flush depends on the terminal configuration. Usually this is at the end of every line, `std::cout` is always flushed before the program reads from `std::cin`. `ofstream` is only flushed when the buffer is full. There is no direct way to flush input streams. 

A steam manipulator `std::flush` allows to control when the steam's buffer is flushed. All the data in the buffer is immediately sent to its destination. This cost some performances, thus should only be used if the data really needs to be up date (e.g. log file to find out why a program crashed). 

