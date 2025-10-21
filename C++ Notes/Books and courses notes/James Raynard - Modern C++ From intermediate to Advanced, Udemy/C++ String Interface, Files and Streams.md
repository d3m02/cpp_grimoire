
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

## Unbuffered Input and Output
The are some applications where stream buffering is not suitable (e.g. a network application, where data must be transmitted in "packet" of a specific size at specific times). 
C++ supports lower level operation on streams to bypass stream's buffer. 
As an example streams member functions `get(char_type& ch)` and `put(char_type ch)` to operate with single character. 
For reading and writing many characters, there are `read(char_type* s, std::streamsize count)` and `write(const char_type* s, std::streamsize count)`, for which we have to provide our own buffer (for `read()` the buffer must be large enough to store all the data we expect to receive). 
Member function `gcount()` return the number of characters that were actually received (it can be used to allocate memory, detect partial or incomplete transfers etc)
```c++
const auto filesize{10U};
char filebuf[filesize];
std::ifstream ifile{"input.txt"};
if (!ifile)
	return -1;

ifile.read(filebuf, filesize);
auto nread = ifile.gcount();
ifile.close();
std::cout << "Read " << nread << " bytes\n";
std::cout.write(filebuf, nread); 
```

#### cppreference note
Mentioned above member functions are part of `std::basic_ostream` and `std::basic_istream` template classes. 
Beside them also exists:
* From `std::basic_istream<CharT,Traits>`:
	* `int_type get();` - reads one character and returns it if available, otherwise returns `Traits::eof()`
	* `basic_istream& get(char_type* s, std::streamsize count, char_type delim = widen('\n'))` - reads characters and stores them into the `s` until in stream not found `delim` or not reached `count` limit, also adds null-character. Almost similar to `read`, but `read` ignores delimiter.
	* `basic_istream& get(basic_streambuf& strbuf, char_type delim = widen('\n'));` - reads characters and inserts them to the output sequence controlled by the given `basic_streambuf` object. 
	* `int_type peek()` - reads the next character from the input stream without extracting it
	* `basic_istream& unget()` - return back to stream last read character, making it available again.
	* `basic_istream& putback(char_type ch)` - put back character `ch` and make it available for next reading. Similar to `unget`, but `unget` don't have an argument. 
	* `basic_istream& ignore(std::streamsize count = 1, int_type delim = Traits::eof());` - extracts and discards characters from the input stream until and including `delim`.
	* `std::streamsize readsome(char_type* s, std::streamsize count);` - read immediately available characters from the input stream and return amount of read characters. Compared to `read()` it's asynchronous non-blokcking call.
* For both `std::basic_ostream<CharT,Traits>` and `std::basic_istream<CharT,Traits>` exists "position" member functions, with suffix p(ut) and g(et) relatively 
	* `pos_type tellp/tellg()` - returns the output position indicator
	* `basic_ostream& seekp/seekg(pos_type pos)` - sets the output absolute position indicator
	* `basic_ostream& seekp/seekg(off_type off, std::ios_base::seekdir dir)` - set position (positive or negative) relative to `dir` base position, `dir` can be `std::ios_base::beg`, `std::ios_base::cur`, `std::ios_base::end`

### File modes 
C++ gives us a number of options for opening a file called "modes".

By default, files are opened in `std::ios_base::in` in "text mode" and output files are opened in `std::ios_base::out` actually opened in `std::ios_base::trunc` "truncate mode" - any previous data will be overwritten starting from the begging of the file.

`std::ios_base::app` will open file in append mode - seek to the end before each write (meaning that `seekp()` have no effect) 
`std::ios_base::binary` - binary mode - the data store in the file will be identical to the data in memory 
`std::ios_base::ate` - will open {a}t {t}he {e}nd, but allows also use of `seekp()`

Modes are actually bit-masks which we can combine for different modes with logical operator "or": `file.open("text.txt", std::ios::out | std::ios::app)`

Restrictions: 
* `std::ios::out` only for `fstream` and `ofstream`
* `std::ios::in` only for `fstream` and `ifstream`
* `std::ios::trunc` only in output mode 
* `std::ios::app` can't combine with `std::ios::trunc`, the file will always be opened in output mode. 
> Note: compiler doesn't provide any error or warnings on restrictions violation and it's classified mostly as undefined behavior

#### cppreference note
Above I used `std::ios_base::in`, `std::ios:in`, also possible to use `std::fstream::in`, `std::ifstream::in`, even `std::ofstream::in`. This due to how inheritance works: 

```mermaid
---
config:
  layout: elk
  look: classic
  theme: dark
---
flowchart TD
    ios_base["std::ios_base"] --> basic_ios["std::basic_ios&lt;CharT, Traits&gt;"]
    basic_ios -- typedef std::basic_ios&lt;char&gt; --> ios(["std::ios"])
    basic_ios -- typedef std::basic_ios&lt;wchar_t&gt; --> wio(["std::wios"])
    basic_ios -- virtual --> basic_istream["std::basic_istream&lt;CharT, Traits&gt;"] & basic_ostream["std::basic_ostream&lt;CharT, Traits&gt;"]
    basic_istream --> basic_iostream["std::basic_iostream&lt;CharT, Traits&gt;"] & basic_ifstream["std::basic_ifstream&lt;CharT, Traits&gt;"]
    basic_ostream --> basic_iostream & basic_ofstream["std::basic_ofstream&lt;CharT, Traits&gt;"]
    basic_istream -- typedef std::basic_istream&lt;char&gt; --> istream(["std::istream"])
    basic_istream -- typedef std::basic_istream&lt;wchar_t&gt; --> wistream(["std::wistream"])
    basic_ostream -- typedef std::basic_ostream&lt;char&gt; --> ostream(["std::ostream"])
    basic_ostream -- typedef std::basic_ostream&lt;wchar_t&gt; --> wostream(["std::wostream"])
    basic_iostream -- typedef std::basic_iostream&lt;char&gt; --> iostream(["std::iostream"])
    basic_iostream -- typedef std::basic_iostream&lt;wchar_t&gt; --> wiostream(["std::wiostream"])
    basic_ifstream -- typedef std::basic_ifstream&lt;char&gt; --> ifstream["std::ifstream"]
    basic_ifstream -- typedef std::basic_ifstream&lt;wchar_t&gt; --> wifstream["std::wifstream"]
    basic_ofstream -- typedef std::basic_ofstream&lt;char&gt; --> ofstream["std::ofstream"]
    basic_ofstream -- typedef std::basic_ofstream&lt;wchar_t&gt; --> wofstream["std::wofstream"]
    basic_iostream --> basic_fstream["std::basic_fstream&lt;CharT, Traits&gt;"]
    basic_fstream -- typedef std::basic_fstream&lt;char&gt; --> fstream(["std::fstream"])
    basic_fstream -- typedef std::basic_fstream&lt;wchar_t&gt; --> wfstream(["std::wfstream"])
    style ios_base fill:#ff6b6b
    style basic_ios fill:#ff9999
    style basic_istream fill:#6bceff
    style basic_ostream fill:#6bceff
    style basic_iostream fill:#6bceff
    style basic_ifstream fill:#90ee90
    style basic_ofstream fill:#90ee90
    style basic_fstream fill:#90ee90
```

In general it's possible to use for any purpose `std::fstream`, but it's not follows [Principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) (use only what is need), can cause performance redundancy (output buffer synchronization when we use file only for reading etc) and actual make some restrictions (in general OS allow multiple descriptors for reading, but since for writing, `fstream` will use both write and read, meaning that where we only read files - we not allow writing in it from other place). 

### Stream Member Functions and State
* `void open(const char* filename, std::ios_base::openmode mode = std::ios_base::out)` - allows to bind file to stream object after it's initialization, for output if file does not already exist, it will be created (similar statement applies to `std::ofstream` constructor as well). `filename` also can be type of `const std::string&` (since C++11) or `const std::filesystem::path::value_type*`/`const std::filesystem::path&` since C++17
* `bool is_open() const` - check if the file is opened.
* `bool good() const` - returns `true` if the input was read successfully.
* `bool fail() const` - returns `true` if there was a recoverable error (e.g. reading in wrong format).
* `bool bad() const` - return `true` if there was an unrecoverable error. 
* `bool clear() const` - restore the stream's state to valid.
* `bool eof() const` - returns true after the end of file has been reached.

One moment about `eof()` 
```c++
std::ifstream ifile{"input.txt"};
int x{};

/* 
Incorrect use of eof if file 
	``
	1
	2
	
	``
- ouput result will be `1, 2, 2` since new line is considered not as end of file, stream will ignore emptyline and not modify x
*/
while (!ifile.eof()) {
	ifile >> x;
	std::cout << x <<", ";
}

/*
"correct way" (unless input not)
	 ``
	 1
	 2
	 abc
	 3
	 ``
- result will be `1, 2` since operator>> will terminate on first non-number character. 
*/ 
while (ifile >> x)
	std::cout << x <<", ";

/* and this where good()/fail() can help to identify incorrect imput*/
```
In next example one of implementation for "correct" input handling. Problem when we input not a number - `oprator>>` terminate execution with setting fail flag, incorrect input stay in buffer (which can cause endless loop). Directly we can't flush input buffer - we can only ignore symbols in it. But we don't known how many symbols to ignore - only way is to ignore maximum possible amount of symbols. But it's also not ideal, input "123a" will output "123"
```c++
int x{};
std::cout << "Input a number: ";
std::cin >> x;

bool success{false};
while (!success) {
    if (std::cin.good()) {
        std::cout << "You entered: " << x << '\n';
        success = true;
    } 
    else if (std::cin.fail()) {
        std::cout << "Invalid input. Please enter a valid number: ";
        std::cin.clear();
        std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    }
}
```
(shorter version)
```c++
int x{};
std::cout << "Input a number: ";

while (!(std::cin >> x)) {
    std::cout << "Invalid input. Please enter a valid number: ";
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
}
```


### Stream Manipulators and Formatting
Manipulator is "something" that gets pushed onto a stream to affect stream's behavior. Some manipulators already presented in `<iostream>` (like `std::endl`, `std::flush`), but there also manipulators from `<iomanip>`
> Actually, manipulators are in headers `<ios>`, `<istream>`, `<ostream>` and `<iomanip>`.

* `std::boolalpha` - print out bool values as "true" and "false", `std::noboolalpha` to disable
* `std::setw(int n)` will pad the next data item on the stream field to make it the width of its argument. By default it right-justified, for switching to left-justified - used `std::left`. 
* `std::left` - _sticky_ manipulator, it will affect whole stream until opposite _sticky_ manipulator not pushed.
* `std::setfill(CharT c)` - sets the padding character. Also _sticky_ (and reset with `std::setfill(' ')`)
```c++
std::cout << std::setfill('.') << std::left 
          << std::setw(15) << "Penguins" << 5 <<'\n'
          <<  "Hippopo" << "tamuses" << '\n'
          << std::setfill(' ') << std::right 
          << std::setw(15) << "Bear" << 2 <<'\n';
          
/*
Penguins.......5
Hippopotamuses
           Bear2
*/
```

### Floating-point Output Formats
C++ by default for relatively small float-point number will output output with 6 significant figures (`std::cout << 3.141'592'653` will display "3.14159"). For large `double` numbers will be provided  scientific notation ("1.00000e-006" represent 1.0000 * 10^-6).

Scientific notation can be force for whole stream with `std::scientific` manipulator. Also with `std::uppercase` 'e' can be forced to 'E' (disabled with `std::nouppercase`). 

The fixed manipulator `std::fixed` will cause floating-point number to be displayed as fixed-point. The number will be displayed to 6 decimal places, truncating or padding with 0 if necessary. Also worth note that `double e{1.602e-19}` with `std::cout << std::fixed << e;` will display "0.000000" since value to small and it's got truncated. 

All mentioned manipulators are _sticky_, meaning that after applying them - unexpectedly stream will remain this changes. Thus it's better to set `std::defaultfloat` after stream being used. 

Manipulator `std::setprecision(int n)` set the precision of the stream(the number of digits that are displayed). 
 
### More from cppreference
Most of manipulators appliable for both input and output streams. 

* `std::dec`, `std::hex`, `std::oct` - set base field for whole stream 
> Note: there no manipulator for binary format, but if it's required - can be used `std::bitset` trick: `std::cout << std::bitset<8>{42}`
* `std::showbase/std::noshowbase` - enable/disable show base for whole stream. Even though it's applies to everything in stream, it have effect only for integer values and when `std::hex` or `std::oct` applied to stream as well.
* `std::showpoint`/`std::noshowpoint` - enable decimal point character for floating-point values. By default it's disable and `std::cout << 3.0` will show just '3', while with `std::showpoint` output will be `3.00000`. Can be combined with `std::setprecision`, but with precision = 2 to achieve output only "3.0" - all other floats will be also affected. 
* `std::showpos`/`std::noshowpos` - shows '+' sign for numeric values (both float and integer)
* `std::skipws`/`std::noskipws` - Enables or disables skipping of leading whitespace by the formatted input functions (enabled by default)
```c++
std::istringstream("a b c") >> c1 >> c2 >> c3;
// c1 = a, c2 = b, c3 = c
std::istringstream("a b c") >> std::noskipws >> c1 >> c2 >> c3;
// c2 = s, c3 = ' ', c3 = b
```
* `std::uppercase`/`std::nouppercase` - enables the use of uppercase characters in floating-point and hexadecimal integer output
* `std::unitbuf`/`std::nounitbuf` - enables or disables automatic flushing of the output stream after any output operation. By default stream flushed when buffer overflow, `std::flush` or `std::endl` pushed, when exiting application. `std::cerr` and `std::wcerr` have `std::unitbuf` enabled. 
* `std::internal` - Sets the adjust field of the stream to internal. Addition to `std::left` and `std::right`.
* `std::hexfloat` - in addition to `std::fixed`, `std::scientific` - change representation of float values. `std::defaultfloat` doesn't affect other float manipulators (like `std::setprecision`)
* `std::ws` - Discards leading whitespace from an input stream. Input-only manipulator. 
* `std::ends` - insert null-symbol. Unlike `std::endl`, does not flush the stream. Typically used with `std::ostrstream`
* `std::emit_on_flush`/`std::noemit_on_flush` - C++20 manipulator for `std::basic_syncbuf`, toggles whether it emits (i.e., transmits data to the underlying stream buffer) when flushed, output-stream only. In reality used not so often, in theory with multiple threads which writes into one stream and required output atomicity, but in most cases it's can be achieved with `std::mutex`/`std::lock_guard`.
* `std::flush_emit` - another C++20 manipulator for `std::basic_syncbuf`, add `buf.emit()` with `os.flush()`, output-only.
* `std::setbase(int base)` - sets the numeric base of the stream, only work with base == 8, 10, 16. 
* `get_money(MoneyT& mon, bool intl = false)`/`put_money(const MoneyT& mon, bool intl = false)`- operate with money format input/output (like "$1,234.5", "2.22 USD", "3.33") and save into integer variable where 2 last symbols - 
* [`std::get_time`](https://en.cppreference.com/w/cpp/io/manip/get_time.html)/[`std::put_time`](https://en.cppreference.com/w/cpp/io/manip/put_time.html) - operate with time format. Quite powerful and flexible thing which support different formats, locals etc, like 
```c++
std::cout.imbue(std::locale("ru_RU.utf8"));
std::cout << "ru_RU: " << std::put_time(&tm, "%c %Z") << '\n';
//ru_RU: Ср. 28 дек. 2011 10:21:16 EST
```
* `std::quoted(s, CharT delim = CharT('"'), CharT escape = CharT('\\')` - allows insertion and extraction of quoted strings. `s` can be type of `const CharT*`, `const std::basic_string<CharT, Traits, Allocator>&`, `std::basic_string_view<CharT, Traits>`, `std::basic_string<CharT, Traits, Allocator>&` (in another words - accept any string). `delim` can be any character, so 
* `std::setiosflags(std::ios_base::fmtflags mask);`/`std::resetiosflags( std::ios_base::fmtflags mask);` - set or unset specific format flag mask. `std::resetiosflags(std::ios_base::basefield)` 