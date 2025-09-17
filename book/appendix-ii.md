Để bạn tiện tham khảo, dưới đây là phần code được tạo ra bởi [script nhỏ mà chúng ta đã viết](representing-code.html#metaprogramming-the-trees) để tự động sinh các class syntax tree cho jlox.

## Expressions

Expressions là những node syntax tree đầu tiên mà ta gặp, được giới thiệu trong "[Representing Code](representing-code.html)". Class `Expr` chính định nghĩa visitor interface dùng để phân phối xử lý đến từng loại expression cụ thể, và chứa các subclass expression khác dưới dạng nested class.

^code expr

### Assign expression

Gán giá trị cho biến được giới thiệu trong "[Statements and State](statements-and-state.html#assignment)".

^code expr-assign

### Binary expression

Các toán tử binary được giới thiệu trong "[Representing Code](representing-code.html)".

^code expr-binary

### Call expression

Biểu thức gọi hàm được giới thiệu trong "[Functions](functions.html#function-calls)".

^code expr-call

### Get expression

Truy cập property, hay "get" expression, được giới thiệu trong "[Classes](classes.html#properties-on-instances)".

^code expr-get

### Grouping expression

Dùng dấu ngoặc đơn để nhóm biểu thức được giới thiệu trong "[Representing Code](representing-code.html)".

^code expr-grouping

### Literal expression

Biểu thức giá trị literal được giới thiệu trong "[Representing Code](representing-code.html)".

^code expr-literal

### Logical expression

Toán tử logic `and` và `or` được giới thiệu trong "[Control Flow](control-flow.html#logical-operators)".

^code expr-logical

### Set expression

Gán giá trị cho property, hay "set" expression, được giới thiệu trong "[Classes](classes.html#properties-on-instances)".

^code expr-set

### Super expression

Biểu thức `super` được giới thiệu trong "[Inheritance](inheritance.html#calling-superclass-methods)".

^code expr-super

### This expression

Biểu thức `this` được giới thiệu trong "[Classes](classes.html#this)".

^code expr-this

### Unary expression

Các toán tử unary được giới thiệu trong "[Representing Code](representing-code.html)".

^code expr-unary

### Variable expression

Biểu thức truy cập biến được giới thiệu trong "[Statements and State](statements-and-state.html#variable-syntax)".

^code expr-variable

## Statements

Statements tạo thành một hệ thống phân cấp syntax tree thứ hai, độc lập với expressions. Chúng ta thêm vài statement đầu tiên trong "[Statements and State](statements-and-state.html)".

^code stmt

### Block statement

Statement block dùng dấu ngoặc nhọn để định nghĩa một scope cục bộ được giới thiệu trong "[Statements and State](statements-and-state.html#block-syntax-and-semantics)".

^code stmt-block

### Class statement

Khai báo class được giới thiệu, không bất ngờ gì, trong "[Classes](classes.html#class-declarations)".

^code stmt-class

### Expression statement

Expression statement được giới thiệu trong "[Statements and State](statements-and-state.html#statements)".

^code stmt-expression

### Function statement

Khai báo hàm được giới thiệu trong "[Functions](functions.html#function-declarations)".

^code stmt-function

### If statement

Câu lệnh `if` được giới thiệu trong "[Control Flow](control-flow.html#conditional-execution)".

^code stmt-if

### Print statement

Câu lệnh `print` được giới thiệu trong "[Statements and State](statements-and-state.html#statements)".

^code stmt-print

### Return statement

Bạn cần một hàm để có thể `return`, nên câu lệnh `return` được giới thiệu trong "[Functions](functions.html#return-statements)".

^code stmt-return

### Variable statement

Khai báo biến được giới thiệu trong "[Statements and State](statements-and-state.html#variable-syntax)".

^code stmt-var

### While statement

Câu lệnh `while` được giới thiệu trong "[Control Flow](control-flow.html#while-loops)".

^code stmt-while