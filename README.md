# Новая архитектура. TO DO: новый readme.md

**В логер добавлены два режима:**

## - Default - с явным выбором режимов запись логов.

```go
func main() {
	logger := createLogger(getRoot())
	logger.Info("App is started!", gologger.OptionConsole(), gologger.OptionFileMutex("log_1"))
	logger.Info("App has terminated!", gologger.OptionConsole(), gologger.GoOptionFileMulti("log_1"))
	time.Sleep(5*time.Second)
}

func createLogger(root string) *gologger.Logger {
	logger := gologger.Default(
		//
		//
		gologger.DefaultConsoleSimple(gologger.BaseLogTemplate),
		//
		//
		gologger.DefaultFileMutex(
			gologger.BaseLogTemplate,
			map[string]string{
				"log_1": root + "/logs/file_1.txt",
			},
		),
		//
		//
		gologger.DefaultFileMulti(
			gologger.BaseLogTemplate,
		),
	)
	return logger
}

```

## - Package - с настройкой записи логов для каждого пакета индивидуально.


```go

func createLogger(root string) *gologger.Logger {
	logger := gologger.Packages(map[string][]gologger.PackageInstaller{
		"main": {
			gologger.PackageConsoleSimple(gologger.BaseLogTemplate, gologger.MultiThreading),
		},
		"mypackage": {
			gologger.PackageConsoleSimple(gologger.BaseLogTemplate, gologger.SingleThreading),
		},
		"/repository/user": {
			gologger.PackageConsoleSimple(gologger.BaseLogTemplate, gologger.MultiThreading),
			gologger.PackageFileMutex(gologger.BaseLogTemplate, gologger.SingleThreading, root + "/logs/file_1.txt"),
		},
		"/usecase/user": {
			gologger.PackageConsoleSimple(gologger.BaseLogTemplate, gologger.MultiThreading),
			gologger.PackageFileMutex(gologger.BaseLogTemplate, gologger.MultiThreading, root + "/logs/file_2.txt"),
		},
	}, )
	return logger
}

func main() {
	logger := createLogger(getRoot())

	mypackage.SetLogs(logger)
	urep.SetLogs(logger)
	ucase.SetLogs(logger)

	mypackage.PrintNumbers(10)
	urep.Log()
	ucase.Log()

	logger.Info("App is started!")
	logger.Info("App has terminated!")
	time.Sleep(5 * time.Second)
}

```

СМ. ПРИМЕРЫ

# gologger - описание | description.

Логгер создавался для того, чтобы одним вызовом функции логгирования, можно было писать сразу в разные накопители.
Одновременная запись, одного лога и в файл и в консоль, с возможностью записывать в отдельном потоке, например только в файл,
а в консоль, только в потоке где была вызвана функция логирования, или вообще все записывать в отдельном потоке,
не заботясь о формате вывода, так как логгер сам создаёт строку вывода в нужном формате.

The logger was created so that with one call to the logging function, it was possible to write to different drives (hard disk, console) at once.
Simultaneous recording of one message to a file and to the console, with the ability to write in a separate stream, for example, only to a file, and to the console, only in the stream where the recording function was called, or write everything in a separate stream, without worrying about the output format, so how the logger itself creates the output string in the desired format.

# Примеры | Examples

Объектом, через которое выполняется логирование является 'UserInterface'. Это объект необходимо создать только один раз в приложении и прокидвать указатель в другие пакеты. 

The object through which logging is performed is 'UserInterface'. This object needs to be created only once in the application and passed the pointer to other packages.

**ПРИМЕР: Создаём глобальный объект, настраиваем его, передаём его дальше. | EXAMPLE: We create a global object, set it up, and pass it on :**
```go
// Создаёт логгер в точке входа.
// Creates a logger at the entry point.

package main

import (
	gologger "../.."
	"./mypackage"
	"time"
)

// Создаём базового логгера, в котором доступен только вывод в консоль.
// We create a basic gologger in which only output to the console is available.
var logs = gologger.Default()

func main() {

	// Выполним логирование, в этом же потоке.
	// Let's perform logging in the same thread.
	logs.Info("App is started!")

	mypackage.SetLogs(logs)
	mypackage.PrintNumbers(10)
	time.Sleep(5 * time.Second)

	logs.Info("App has terminated!")
}

```
**ПРИМЕР: Получаем глобальный объект в других пакетах. | EXAMPLE: Getting the global object in other packages :**
```go

// Передаём объект логгера в другие пакеты.
// We transfer the logger object to other packages.

package mypackage

import (
	gologger "../../.."
	"strconv"
)

var (
	logs *gologger.LogInterface
)

func SetLogs(main *gologger.LogInterface) {
	logs = main
}
```

## Базовая настройка. | Basic setting. 

В базовой настройке доступен только вывод в консоль. Все остальные настройки устанавливаются после получения базовой настройки.

In the basic setting, only console output is available. All other settings are set after receiving the basic setting.

**ПРИМЕР: Базовая настройка | EXAMPLE: Basic setting :**
```go

// Создаём базового логгера, в котором доступен только вывод в консоль.
// We create a basic gologger in which only output to the console is available.
var logs = gologger.Default()

func main() {

	// Выполним логирование, в этом же потоке.
	// Let's perform logging in the same thread.
	logs.Info("App is started!")
	
	logs.Info("App has terminated!")
}

[OUTPUT]:
2020/08/20 10:46:47 level=[INFO];func=[main.main];value=["App is started!"];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:52 level=[INFO];func=[main.main];value=["App has terminated!"];date=[Thu Aug 20 10:46:52 2020];

```

Для пользовательского интерфейса используется паттерн программирования 'Functional options'. Опции ('Option') выстыпают в качестве функций, которые возвращают функции 'Mode'. Функции 'Mode', в свою очередь вызывают метод 'add' у конкретного логгера реализующего интерфейс 'iLogger' (объект хранится в глобальном пользовательском объекте 'LogInterface'). Обычно этот паттерн используется для создания новых объектов, но в данном случае только выбора вывода лога.

The user interface uses the 'Functional options' programming pattern. Options ('Option') pop out as functions that return 'Mode' functions. The 'Mode' functions, in turn, call the 'add' method of a specific logger that implements the 'iLogger' interface (the object is stored in the global user object 'LogInterface').Usually this pattern is used to create new objects, but in this case only to select the log output.

**ТАК РАБОТАЕТ: Сигнатура функции настройки вывода | IMPLEMENTATION: Output customization function signature :**

```go
// Mode : в теле содержит вызов метода 'add' конкретного логгера.
//        in the body contains a call to the 'add' method of a particular logger.
//
type Mode func(logger *LogInterface, value interface{}, lvl level, date string)

// Option : возвращает 'Mode' соответствующий  выбранной пользователем опции.
//          returns 'Mode' corresponding to the option selected by the user.
//
type Option func(param ...string)

// 'Option' which return 'Mode'
func Console(param ...string) Mode {
	return func(logger *LogInterface, value interface{}, lvl level, date string) {
		logger.modeConsole.add(value, lvl, date, param...)
	}
}

// 'Option' which return 'Mode'
func GoConsole(param ...string) Mode {
	return func(logger *LogInterface, value interface{}, lvl level, date string) {
		go logger.modeConsole.add(value, lvl, date, param...)
	}
}

```

Список функций настройки вывода ('Option'):

- gologger.Console : возвращает 'Mode' соответствующий 'loggerConsoleSinglethreading'. Вызов в том же потоке.

- gologger.GoConsole : возвращает 'Mode' соответствующий 'loggerConsoleSinglethreading'. Вызов в отдельном потоке.

- gologger.FileMulti : возвращает 'Mode' соответствующий 'loggerFileMultithreading'. Вызов в том же потоке.

- gologger.GoFileMulti : возвращает 'Mode' соответствующий 'loggerFileMultithreading'. Вызов в отдельном потоке.

- gologger.FileMutex : возвращает 'Mode' соответствующий 'loggerFileMutex'. Вызов в том же потоке.

- gologger.GoFileMutex : возвращает 'Mode' соответствующий 'loggerFileMutex'. Вызов в отдельном потоке.


Output Setting ('Option') Function List:

- gologger.Console: returns 'Mode' corresponding to 'loggerConsoleSinglethreading'. Call on the same thread.

- gologger.GoConsole: returns 'Mode' corresponding to 'loggerConsoleSinglethreading'. Call in a separate thread.

- gologger.FileMulti: returns 'Mode' corresponding to 'loggerFileMultithreading'. Call on the same thread.

- gologger.GoFileMulti: returns 'Mode' corresponding to 'loggerFileMultithreading'. Call in a separate thread.

- gologger.FileMutex: returns 'Mode' corresponding to 'loggerFileMutex'. Call on the same thread.

- gologger.GoFileMutex: returns 'Mode' corresponding to 'loggerFileMutex'. Call in a separate thread.


**ПРИМЕР: Вывод в отдельном потоке. | EXAMPLE: Output in a separate thread :**
```go

func PrintNumbers(num int) {

	prev := new(number)

	for i := 0; i < num; i++ {

		currentNumber := &number{
			Value: i,
			Prev:  prev,
		}

		// Распечатаем сообщение в отдельном потоке.
		// Let's print the message in a separate thread.
		logs.Info(currentNumber, gologger.GoConsole())

		prev = currentNumber
	}

	logs.Info("Print all numbers : " + strconv.Itoa(num))
}

[OUTPUT]:
2020/08/20 10:46:47 level=[INFO];func=[main.main];value=["App is started!"];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=["Print all numbers : 10"];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":0,"Prev":{"Value":0,"Prev":null}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":6,"Prev":{"Value":5,"Prev":{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":8,"Prev":{"Value":7,"Prev":{"Value":6,"Prev":{"Value":5,"Prev":{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":7,"Prev":{"Value":6,"Prev":{"Value":5,"Prev":{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":9,"Prev":{"Value":8,"Prev":{"Value":7,"Prev":{"Value":6,"Prev":{"Value":5,"Prev":{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}}}}}}];date=[Thu Aug 20 10:46:47 2020];
2020/08/20 10:46:47 level=[INFO];func=[mypackage.PrintNumbers];value=[{"Value":5,"Prev":{"Value":4,"Prev":{"Value":3,"Prev":{"Value":2,"Prev":{"Value":1,"Prev":{"Value":0,"Prev":{"Value":0,"Prev":null}}}}}}}];date=[Thu Aug 20 10:46:47 2020];

```
## Настройка для работы с файлами. | Setting for working with files.

Расширяем настройки базового 'LogInterface' для работы с файлами.

Expanding the basic 'LogInterface' settings for working with files.

**ПРИМЕР: Настройка для работы с файлами. | EXAMPLE: Setting for working with files :**

```go

package main

import (
	gologger "../.."
	"path/filepath"
	"runtime"
)

var (
	_, b, _, _   = runtime.Caller(0)
	projectRoot  = filepath.Dir(b)
	loggingFiles = map[string]string{
		"file_1": projectRoot + "/log1.txt",
		"file_2": projectRoot + "/log2.txt",
	}
)

func main() {
	logs := settings(loggingFiles)
}

func settings(files map[string]string) *gologger.LogInterface {

	// Создаём базового логгера. Добавляем файлы для логирования.
	// We create a basic logger. Add files for logging.
	logs := gologger.Default().ConfigFile(files)

	// Устанавливаем режим запись через каналы.
	// We set the recording mode through channels.
	logs.SetModeFileMulti()

	// Устанавливаем режим записи через логгер из стандартной библиотеки
	// We set the recording mode through the logger from the standard library
	logs.SetModeFileMutex()

	return logs
}
```

**ПРИМЕР: Запись в разные накопители. | EXAMPLE: Writing to different drives :**
```go
func main() {
	scanner := bufio.NewScanner(os.Stdin)

	logs := settings(loggingFiles)

	logs.Info("Logger wes set!")

	// Выполним логирование, всеми установленными способами, в этом же потоке,
	// кроме вывода в консоль, его выполним в отдельном потоке.
	// Let's perform logging, in all the established ways, in the same thread,
	// except for output to the console, we will execute it in a separate thread.
	logs.Info("App is Started!", gologger.GoConsole(), gologger.FileMulti("file_1"), gologger.FileMutex("file_2"))

	for scanner.Scan() {

		message := scanner.Text()

		if message == "" {

			// Выполним логирование в консоль, в том же потоке, а в файл с помощью каналов (GoFileMulti), выполним в отдельном потоке.
			// Let's log into the console, in the same thread, and to a file using channels (GoFileMulti), we'll execute it in a separate thread.
			logs.Error("Message is empty", gologger.Console(), gologger.GoFileMulti("file_1"))
		} else {

			// Выполним логирование в консоль и в файл, с помощью стандартного логгера (GoFileMutex), выполним в отдельном потоке.
			// Let's log into the console and into a file using a standard logger (GoFileMutex), and execute it in a separate thread.
			logs.Info(message, gologger.GoConsole(), gologger.GoFileMutex("file_2"))
		}
	}
}

[OUTPUT]:
2020/08/20 12:25:24 level=[INFO];func=[];value=["LogInterface message : from 'SetModeFileMulti()' mode was set"];date=[Thu Aug 20 12:25:24 2020];
2020/08/20 12:25:24 level=[INFO];func=[];value=["LogInterface message : from 'SetModeFileMutex()' mode was set"];date=[Thu Aug 20 12:25:24 2020];
2020/08/20 12:25:24 level=[INFO];func=[main.main];value=["Logger wes set!"];date=[Thu Aug 20 12:25:24 2020];
2020/08/20 12:25:24 level=[INFO];func=[main.main];value=["App is Started!"];date=[Thu Aug 20 12:25:24 2020];

2020/08/20 12:25:33 level=[ERROR];func=[main.main];value=["Message is empty"];date=[Thu Aug 20 12:25:33 2020];
Hello world!
2020/08/20 12:25:42 level=[INFO];func=[main.main];value=["Hello world!"];date=[Thu Aug 20 12:25:42 2020];
Goodbye world!
2020/08/20 12:25:51 level=[INFO];func=[main.main];value=["Goodbye world!"];date=[Thu Aug 20 12:25:51 2020];

[file_1]:
level=[INFO];func=[main.main];value=["App is Started!"];date=[Thu Aug 20 12:25:24 2020];
level=[ERROR];func=[main.main];value=["Message is empty"];date=[Thu Aug 20 12:25:33 2020];

[file_2]:
2020/08/20 12:25:24 level=[INFO];func=[main.main];value=["App is Started!"];date=[Thu Aug 20 12:25:24 2020];
2020/08/20 12:25:42 level=[INFO];func=[main.main];value=["Hello world!"];date=[Thu Aug 20 12:25:42 2020];
2020/08/20 12:25:51 level=[INFO];func=[main.main];value=["Goodbye world!"];date=[Thu Aug 20 12:25:51 2020];
```

# Особенности | Features.

Для записи в файл существует две реализации:

- через каналы.

- с помощью стандартного пакета 'log'.

## Запись в файл через каналы.

Особенностью этого решения является то, что для каждого из файлов создаётся буфферизированный канал на 1000 элементов (строк, которые надо записать в файл).
Для каждого такого канала, запускается в отдельном потоке горутина-читатель, которая имеет право вызвать функцию записи в файл,
что гарантирует то, что не возникнет ситуация гонки. Буффер на 1000 элементов теоритически достаточно большой, для того чтобы горутины писатели не вставали в очередь
на запись в буффер канала.

A feature of this solution is that for each of the files, a buffered channel is created for 1000 elements (lines that must be written to the file).
For each such channel, a reader goroutine is launched in a separate thread, which has the right to call the function of writing to the file, which guarantees that a race situation does not arise. The 1000-element buffer is theoretically large enough to prevent writers from queuing up to write to the channel buffer.

**ТАК РАБОТАЕТ: Горутина-читатель | IMPLEMENTATION: goroutine-reader :**
```go
func (logger *loggerFileMultithreading) receiver(file fileAgent) {
	for outputString := range file.channel {
		runtime.Gosched()
		err := logger.output(outputString, file.path)
		if err != nil {
			logger.errorOutput(outputString, err)
		}
	}
}
```

После открытия файла, создаёт временный буффер,в который будет выполняться запись.Следом, вызывается системный вызов fsync(),для сборса буферов файловой системы на диск.
После записи содержимого в буффер, данные сбрасываются в файл через '(*io.Writer).Flush()'.

After opening the file, it creates a temporary buffer to write to. Next, the fsync() system call is called to collect the file system buffers to disk.
After writing the content to the buffer, the data is flushed to the file via '(*io.Writer).Flush()'.

**ТАК РАБОТАЕТ: Запись в файл | IMPLEMENTATION: Write to file :**
```go
func (logger *loggerFileMultithreading) output(out *outputString, param ...string) error {
	var (
		path      = param[0]
		closeFile = func(file *os.File, err error) error {
			errFile := file.Close()
			if errFile != nil {
				err = errors.New(strings.Join([]string{
					err.Error(),
					errFile.Error(),
				}, "::"))
			}
			return err
		}
	)
	file, err := os.OpenFile(path, os.O_APPEND, 0666)
	if err != nil {
		return err
	}
	buffer := bufio.NewWriter(file)
	err = file.Sync()
	if err != nil {
		return closeFile(file, err)
	}
	_, err = buffer.WriteString(string(*out) + "\n")
	if err != nil {
		return closeFile(file, err)
	}
	err = buffer.Flush()
	if err != nil {
		return closeFile(file, err)
	}
	return closeFile(file, nil)
}
```

![alt text](https://github.com/RobertGumpert/gologger/blob/master/examples/channel.png)


## Запись с помощью стандартного пакета 'log'.

Используется стандартный пакет 'log', который разрешает состояние гонки с помощью мьютексов.

The standard package 'log' is used, which resolves race conditions using mutexes.

**ТАК РАБОТАЕТ: Запись в файл | IMPLEMENTATION: Write to file :**
```go
func (logger *loggerFileMutex) output(out *outputString, param ...string) error {
	var (
		path      = param[0]
		closeFile = func(file *os.File, err error) error {
			errFile := file.Close()
			if errFile != nil {
				err = errors.New(strings.Join([]string{
					err.Error(),
					errFile.Error(),
				}, "::"))
			}
			return err
		}
	)
	file, err := os.OpenFile(path, os.O_APPEND, 0666)
	if err != nil {
		return closeFile(file, nil)
	}
	log.SetOutput(file)
	log.Print(*out)
	return closeFile(file, nil)
}
```
