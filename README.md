# qt-bison-flex

This is a sample project demonstrating the integration of a Flex/Bison parser into a Qt GUI application. Since code demonstration is the main goal of this project, no binaries are included. The project can be opened and compiled with [Qt Creator][creator]. A detailed description of the code is provided [below](#about).

## Setup

Since [kits][kits], compiler paths etc. are almost certainly different between systems, the project does not include the `.pro.user` file. After opening the project, go to the **Projects** tab in Creator and set up your kit(s). You will need to add two custom process steps to the compilation to make sure the lexer and parser are regenerated should you make any changes to Flex or Bison source files:

 * **Custom process step**
   * Command: `bison`
   * Argument: `parser.y`
   * Working directory: `%{sourceDir}`
 * **Custom process step**
   * Command: `flex`
   * Argument: `scanner.l`
   * Working directory: `%{sourceDir}`

Compiling with GCC 5.4.0 on Xubuntu 16.04 _should_ return with only one warning (a `-Wsign-compare` in `scanner.cpp`, line 1102). Your mileage may vary.

## Input files

Once compiled, the executable needs an input file to parse. The simple -- and rather nonsensical -- "Sample input file" (`.sif`) format consists of:

 * An opening line containing `HELLO`
 * Any number of specification lines of the format `max number align alignment`, where `number` must be an integer and `alignment` is either `left`, `right` or `center`.
 * A closing line containing `GOODBYE`
 
For each specification line, the program creates two widgets in the "Parse results" frame: a spinbox from 0 to the number given after `max`, and a label with the text "Sample text" and the alignment specified (left, right or center).

## About

This description is not meant to be an exhaustive tutorial on Bison/Flex or Qt, but a general description of the approach I used for integrating them.

### Making an `#include`able lexer/parser

Bison and Flex need to be told to generate header files, so that they can be `#include`ed in other parts of the project. This can be accomplished by command line parameters to the `flex` and `bison` calls, or (my preferred method) using directives inside the source files. Here's an overview of the options I used in this project:

- **Bison ([`parser.y`](parser.y))**
  - Output file: [`parser.cpp`](parser.cpp) (use `%output "parser.cpp"`)
  - Header file: [`parser.h`](parser.h) (use `%defines "parser.h"`)
- **Flex ([`scanner.l`](scanner.l))**
  - The first section (between `%{` and `%}`) must contain an `#include` to the header file generated by Bison (in this case `parser.h`)
  - Output file: [`scanner.cpp`](scanner.cpp) (use `%option outfile="scanner.cpp"`)
  - Header file: [`scanner.h`](scanner.h) (use `%option header-file="scanner.h"`)

Note that specifying the output file names is not required, it was only used for consistency (i.e. matching base names, as well as `.cpp` extensions as opposed to the default `.c`). As mentioned in [Setup](#setup), Bison and Flex must be run (in that order) to generate the lexer/parser:

```
bison parser.y
flex scanner.l
```

The lexer/parser can be used by including the header files at the desired point in the project.

### The Flex source

As mentioned [above](#making-an-includeable-lexerparser), the top section needs to `#include` the Bison-generated header. The only other include in this project is `<cstdlib>`, as char-to-integer conversion is done by `atoi()`. The `yylex()` and `yyerror()` declarations are basically boilerplate. The `noyywrap` and `nounput` options are used to suppress compiler warnings and to avoid having to declare a "dummy" `yywrap()` function (or macro).

### The Bison source

Bison can make use of any Qt class, although this is somewhat obscured by having all declarations in [`common.h`](common.h) ( e.g. `elements_t` is a `typedef` for a `QVector` of `ParseElement` objects). This seems to be restricted, however, to actual C++ blocks, i.e. the top and bottom sections as well as any (brace-enclosed) actions within rules. The `%union` directive, for example, would not accept `Qt::AlignmentFlag`, which is why that type is passed between rules by `static_cast`ing to/from `int`.

In case of an error, `yyerror()` throws an exception and stops parsing (the exception propagates up and is caught by the `try` block surrounding the call to `yyparse()` in [`testparser.cpp`](testparser.cpp)). This was done to keep this demo simple, however, there are possibilities for more sophisticated error handling and reporting. For example, while you can’t use `emit` within the Bison source (since it won't be processed by the [meta-object compiler][moc]), you can create a `QObject` with proxy methods emitting signals, and call those from the actions. By using the [`qRegisterMetatype<>()`][qregistermetatype] method, any\* object can be sent through the signal/slot mechanism, so you could, say, emit a signal with every successfully parsed item.

\**From the [docs][qregistermetatype]: "Any class or struct that has a public default constructor, a public copy constructor and a public destructor can be registered."*

### Interacting with Qt

Running `bison` and `flex` will generate the header and source files, and by including the header files we now have access to the lexer/parser. However, there are still some not-quite-object-oriented aspects that need encapsulation. This is done in the `TestParser` class ([`testparser.h`](testparser.h), [`testparser.cpp`](testparser.cpp)).

`TestParser`'s constructor takes the file contents as a `QByteArray`. This way, we can make use of Qt's platform-independent facilities for opening and reading files (not to mention presenting a user with a dialog to pick the desired input file). The file contents could be preprocessed (e.g. to ensure proper encoding), or they need not even be read from a file: you could present the user with a `QTextEdit`, let them type in the code, then parse it without having to open a file.

In order for Bison to take string input, the function `yy_scan_string()` must be used, returning a `YY_BUFFER_STATE` which must be freed using `yy_delete_buffer()`. To this end, the simple `YYBufferGuard` helper class is defined. After creating the `YY_BUFFER_STATE`, we pass a pointer to it to the `YYBufferGuard` constructor, and rely on automatic storage duration to invoke its destructor, which calls `yy_delete_buffer()`. The `Q_UNUSED` macro suppresses the `-Wunused-variable` warning generated when the compiler sees that the guard object is used only once, during its instantiation.

You can access any objects or functions declared in your Bison source by putting an `extern` declaration in the source file (for functions, use `extern "C"`). This is how the parsed elements are retreived from the parser. I've assembled a (probably non-exhaustive) list of approaches for moving data between the parser and the "outside world" [here](#exchanging-data-with-the-parser).

You might notice that `TestParser` does not derive from `QObject`. This is only because of the simplicity of this demo app did not require it: we instantiate a TestParser passing the contents of the input file as a `QByteArray`, then access a vector containing the parsed elements (which will be empty if parsing fails). Deriving from `QObject` should not impact your parser in any way.

### The GUI

Since the GUI was not the primary focus, I kept it as simple as possible. Ordinarily, I would use a separate top-level application object that manages the GUI and the business logic separately from each other, but in this case, the logic code was small enough that it could be included directly into `MainWindow`'s code.

During the instantiation of the UI class, the [meta-object compiler][moc] automatically invokes the [`QMetaObject::connectSlotsByName()`][connectslotsbyname] method. As the name suggests, it checks for slots with a suitable name and signature, and connects them to the corresponding signals. The GUI contains two buttons named `browseButton` and `parseButton` (see [`mainwindow.ui`](mainwindow.ui)), whose `clicked` signals will be connected to `on_browseButton_clicked()` and `on_parseButton_clicked()`, respectively.

The `resetResultWidgets()` method removes all widgets from the result area. This is not as straightforward as it may seem: each item is retreived as a `QLayoutItem`, the corresponding `QWidget` is deleted through the layout item's `widget()` pointer, after which the `QLayoutItem` itself is deleted.

When the user clicks the "Parse" button, the `MainWindow::on_parseButton_clicked()` method is invoked. If a file path has been selected, it opens the file, reads its contents into a `QByteArray` and creates a `TestParser` using that data. The `TestParser` constructor takes the data, passes it to the parser (using `QByteArray::constData()` to convert it into a `const char*`). Regardless of the outcome, a `TestParser` object will be instantiated, providing the `elements()` functions returning the parsed elements (`elements_t`, i.e. `QVector<ParseElement>`).

If the parsing fails, no elements are returned, though this was purely a matter of choice. If you design your parser so that it never creates invalid elements, your could return all elements parsed up to the point where the error was encountered.

### Exchanging data with the parser

#### Getting data into the parser:
- Through variables:
 + Declare a variable in the Bison source
 + `extern` declare it in a source file that includes the Bison header
 + Set the value of the variable from outside the parser
- Through setter functions:
 + Define a setter function in the Bison source
 + `extern "C"` declare it in a source file that includes the Bison header
 + Invoke the setter function from outside the parser

#### Getting data out of the parser:
- Through variables:
 + Declare a variable in the Bison source
 + `extern` declare it in a source file that includes the Bison header
 + Set the value in the parser
- Through a helper class:
 + Define a class to hold your data, with setter functions
 + Include the header in the Bison source file
 + Instantiate the helper object in the parser
 + `extern` declare the helper object outside the parser
 + Invoke the setter functions in the parser
- Through Qt signals:
 + Define a `QObject`-based helper class with functions that emit signals
 + Instantiate it outside the parser
 + Connect the signals
 + Pass a pointer to the helper class to the parser
 + Invoke the emitter functions in the parser.

[creator]: http://doc.qt.io/qtcreator/index.html
[kits]: http://doc.qt.io/qtcreator/creator-targets.html
[moc]: http://doc.qt.io/qt-5/moc.html
[qregistermetatype]: http://doc.qt.io/qt-5/qmetatype.html#qRegisterMetaType
[connectslotsbyname]: http://doc.qt.io/qt-5/qmetaobject.html#connectSlotsByName
