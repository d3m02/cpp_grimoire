#cpp17 
`Optional data` – решение, когда существует переменная, которая может оказаться пустой. К примеру, если требуется считать файл, возможна ситуация, когда чтение будет неудачным и в переменную ничего не будет помещено. В случае не успешного чтения из файла – встает вопрос, что делать с этим. Можно вернуть пустую строку, но это тоже не однозначное решение, возможно файл существует и он просто пустой, что не является ошибкой. 
```c++
std::optional<std::string>ReadFile (const std::string& path)
{
    std::ifstram stream (path);
    if (!stream)
	    return {};  // nothing/no data
    
    std::string res;
    // something
    std:stream.close();
    return res;
}

std::optinal<std::string>data = ReadFile ("data.txt");
std::string value = data.value_or ("Not present");

if (data.has_value())
//или if (data)
{  
 // something
}
```