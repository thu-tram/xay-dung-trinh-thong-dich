Dưới đây là toàn bộ grammar của Lox. Các chương giới thiệu từng phần của ngôn ngữ đều có kèm các quy tắc grammar tương ứng, nhưng ở đây chúng được tập hợp lại thành một chỗ.

## Syntax Grammar

**Syntax grammar** được dùng để phân tích chuỗi token tuyến tính thành cấu trúc cây cú pháp lồng nhau. Nó bắt đầu với quy tắc đầu tiên khớp với toàn bộ một chương trình Lox (hoặc một mục nhập đơn trong REPL).

```ebnf
program        → declaration* EOF ;
```

### Declarations

Một chương trình là một chuỗi các **declaration**, tức là các statement gắn kết định danh mới hoặc bất kỳ loại statement nào khác.

```ebnf
declaration    → classDecl
               | funDecl
               | varDecl
               | statement ;

classDecl      → "class" IDENTIFIER ( "<" IDENTIFIER )?
                 "{" function* "}" ;
funDecl        → "fun" function ;
varDecl        → "var" IDENTIFIER ( "=" expression )? ";" ;
```

### Statements

Các quy tắc statement còn lại tạo ra **side effect**, nhưng không giới thiệu binding mới.

```ebnf
statement      → exprStmt
               | forStmt
               | ifStmt
               | printStmt
               | returnStmt
               | whileStmt
               | block ;

exprStmt       → expression ";" ;
forStmt        → "for" "(" ( varDecl | exprStmt | ";" )
                           expression? ";"
                           expression? ")" statement ;
ifStmt         → "if" "(" expression ")" statement
                 ( "else" statement )? ;
printStmt      → "print" expression ";" ;
returnStmt     → "return" expression? ";" ;
whileStmt      → "while" "(" expression ")" statement ;
block          → "{" declaration* "}" ;
```

Lưu ý rằng `block` là một quy tắc statement, nhưng cũng được dùng như một nonterminal trong một vài quy tắc khác, chẳng hạn như thân hàm.

### Expressions

**Expression** tạo ra giá trị. Lox có nhiều toán tử unary và binary với các mức độ ưu tiên khác nhau. Một số grammar của ngôn ngữ không mã hóa trực tiếp quan hệ ưu tiên mà quy định ở nơi khác. Ở đây, chúng ta dùng một quy tắc riêng cho mỗi mức ưu tiên để thể hiện rõ ràng.

```ebnf
expression     → assignment ;

assignment     → ( call "." )? IDENTIFIER "=" assignment
               | logic_or ;

logic_or       → logic_and ( "or" logic_and )* ;
logic_and      → equality ( "and" equality )* ;
equality       → comparison ( ( "!=" | "==" ) comparison )* ;
comparison     → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
term           → factor ( ( "-" | "+" ) factor )* ;
factor         → unary ( ( "/" | "*" ) unary )* ;

unary          → ( "!" | "-" ) unary | call ;
call           → primary ( "(" arguments? ")" | "." IDENTIFIER )* ;
primary        → "true" | "false" | "nil" | "this"
               | NUMBER | STRING | IDENTIFIER | "(" expression ")"
               | "super" "." IDENTIFIER ;
```

### Utility rules

Để giữ cho các quy tắc trên gọn gàng hơn, một số phần của grammar được tách ra thành các quy tắc phụ tá có thể tái sử dụng.

```ebnf
function       → IDENTIFIER "(" parameters? ")" block ;
parameters     → IDENTIFIER ( "," IDENTIFIER )* ;
arguments      → expression ( "," expression )* ;
```

## Lexical Grammar

**Lexical grammar** được scanner dùng để nhóm các ký tự thành token. Nếu syntax là [context free](https://en.wikipedia.org/wiki/Context-free_grammar), thì lexical grammar là [regular](https://en.wikipedia.org/wiki/Regular_grammar) — lưu ý rằng không có quy tắc đệ quy.

```ebnf
NUMBER         → DIGIT+ ( "." DIGIT+ )? ;
STRING         → "\"" <any char except "\"">* "\"" ;
IDENTIFIER     → ALPHA ( ALPHA | DIGIT )* ;
ALPHA          → "a" ... "z" | "A" ... "Z" | "_" ;
DIGIT          → "0" ... "9" ;
```