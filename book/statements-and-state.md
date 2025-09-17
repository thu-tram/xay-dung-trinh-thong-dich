> Suốt cuộc đời mình, trái tim tôi luôn khao khát một điều mà tôi không thể gọi tên.
> <cite>Andr&eacute; Breton, <em>Mad Love</em></cite>

Interpreter mà ta có cho đến giờ cảm giác giống như đang bấm nút trên một chiếc máy tính bỏ túi hơn là lập trình một ngôn ngữ thực thụ. “Lập trình” với tôi nghĩa là xây dựng một hệ thống từ những mảnh ghép nhỏ hơn. Ta chưa thể làm điều đó vì chưa có cách nào để gán một tên cho một dữ liệu hoặc hàm. Ta không thể ghép nối phần mềm nếu không có cách tham chiếu tới các mảnh ghép.

Để hỗ trợ việc gán tên (binding), interpreter của ta cần có trạng thái bên trong. Khi bạn định nghĩa một biến ở đầu chương trình và sử dụng nó ở cuối, interpreter phải giữ lại giá trị của biến đó trong suốt thời gian đó. Vậy nên trong chương này, ta sẽ cho interpreter một “bộ não” không chỉ biết xử lý, mà còn biết *ghi nhớ*.

<img src="image/statements-and-state/brain.png" alt="Một bộ não, có lẽ đang ghi nhớ gì đó." />

State và <span name="expr">statement</span> luôn đi cùng nhau. Vì statement, theo định nghĩa, không trả về giá trị, nên chúng cần làm điều gì khác để hữu ích. Điều đó được gọi là **side effect** (tác dụng phụ). Nó có thể là tạo ra đầu ra mà người dùng nhìn thấy hoặc thay đổi một trạng thái nào đó trong interpreter để có thể được phát hiện sau này. Cách thứ hai khiến chúng rất phù hợp để định nghĩa biến hoặc các thực thể có tên khác.

<aside name="expr">

Bạn hoàn toàn có thể tạo ra một ngôn ngữ coi khai báo biến là một expression vừa tạo binding vừa trả về giá trị. Ngôn ngữ duy nhất tôi biết làm vậy là Tcl. Scheme có vẻ cũng gần giống, nhưng lưu ý rằng sau khi một expression `let` được đánh giá, biến mà nó bind sẽ bị quên đi. Cú pháp `define` không phải là một expression.

</aside>

Trong chương này, ta sẽ làm tất cả những điều đó. Ta sẽ định nghĩa các statement tạo ra đầu ra (`print`) và tạo state (`var`). Ta sẽ thêm expression để truy cập và gán giá trị cho biến. Cuối cùng, ta sẽ thêm block và scope cục bộ. Nghe có vẻ nhiều cho một chương, nhưng ta sẽ xử lý từng phần một.

## Statement

Ta bắt đầu bằng cách mở rộng grammar của Lox với statement. Chúng không khác nhiều so với expression. Ta sẽ bắt đầu với hai loại đơn giản nhất:

1.  **Expression statement** cho phép bạn đặt một expression ở vị trí mà một statement được mong đợi. Chúng tồn tại để đánh giá các expression có side effect. Bạn có thể không để ý, nhưng bạn dùng chúng mọi lúc trong <span name="expr-stmt">C</span>, Java và nhiều ngôn ngữ khác. Bất cứ khi nào bạn thấy một lời gọi hàm hoặc phương thức theo sau bởi `;`, đó chính là một expression statement.

    <aside name="expr-stmt">

    Pascal là một ngoại lệ. Nó phân biệt giữa *procedure* và *function*. Function trả về giá trị, còn procedure thì không. Có một dạng statement để gọi procedure, nhưng function chỉ có thể được gọi ở nơi một expression được mong đợi. Pascal không có expression statement.

    </aside>

2.  **`print` statement** đánh giá một expression và hiển thị kết quả cho người dùng. Tôi thừa nhận rằng việc “nướng” tính năng in ra ngay trong ngôn ngữ thay vì biến nó thành một hàm thư viện là hơi lạ. Nhưng đây là sự nhượng bộ trước thực tế rằng ta đang xây dựng interpreter này từng chương một và muốn có thể thử nghiệm với nó trước khi hoàn thiện. Để biến print thành một hàm thư viện, ta sẽ phải đợi cho đến khi có đầy đủ cơ chế định nghĩa và gọi hàm <span name="print">trước</span> khi có thể chứng kiến bất kỳ side effect nào.

    <aside name="print">

    Tôi chỉ xin nói một chút để bảo vệ rằng BASIC và Python có `print` statement riêng và chúng là những ngôn ngữ thực thụ. Tất nhiên, Python đã bỏ `print` statement trong phiên bản 3.0…

    </aside>

Cú pháp mới đồng nghĩa với các quy tắc grammar mới. Trong chương này, cuối cùng ta cũng có khả năng parse toàn bộ một script Lox. Vì Lox là một ngôn ngữ imperative, kiểu động, “tầng trên cùng” của một script đơn giản là một danh sách statement. Các quy tắc mới là:

```ebnf
program        → statement* EOF ;

statement      → exprStmt
               | printStmt ;

exprStmt       → expression ";" ;
printStmt      → "print" expression ";" ;
```

Quy tắc đầu tiên giờ là `program`, điểm bắt đầu của grammar và đại diện cho một script Lox hoàn chỉnh hoặc một mục nhập trong REPL. Một program là danh sách các statement theo sau bởi token đặc biệt “end of file”. Token kết thúc bắt buộc này đảm bảo parser tiêu thụ toàn bộ input và không âm thầm bỏ qua các token thừa lỗi ở cuối script.

Hiện tại, `statement` chỉ có hai trường hợp cho hai loại statement mà ta vừa mô tả. Ta sẽ bổ sung thêm sau trong chương này và các chương tiếp theo. Bước tiếp theo là biến grammar này thành thứ mà ta có thể lưu trữ trong bộ nhớ — syntax tree.


### Syntax Tree (Cây cú pháp) cho Statement

Không có chỗ nào trong grammar mà cả expression và statement đều được phép xuất hiện. Toán hạng của, ví dụ, `+` luôn là expression, không bao giờ là statement. Thân của một vòng lặp `while` thì luôn là một statement.

Vì hai cú pháp này tách biệt, ta không cần một base class chung mà tất cả đều kế thừa. Việc tách expression và statement thành hai hệ phân cấp class riêng cho phép trình biên dịch Java giúp ta phát hiện những lỗi ngớ ngẩn như truyền một statement vào một phương thức Java đang mong đợi một expression.

Điều đó có nghĩa là ta cần một base class mới cho statement. Như các bậc tiền bối đã làm trước đây, ta sẽ dùng cái tên “Stmt” đầy bí ẩn. Với tầm nhìn <span name="foresight">xa</span>, tôi đã thiết kế script metaprogramming AST nhỏ của chúng ta để chuẩn bị cho việc này. Đó là lý do ta đã truyền “Expr” làm tham số cho `defineAst()`. Giờ ta thêm một lời gọi khác để định nghĩa Stmt và các <span name="stmt-ast">subclass</span> của nó.

<aside name="foresight">

Thực ra cũng chẳng phải tầm nhìn gì: tôi đã viết toàn bộ code cho cuốn sách trước khi chia nó thành các chương.

</aside>

^code stmt-ast (2 before, 1 after)

<aside name="stmt-ast">

Code được generated cho các node mới nằm ở [Phụ lục II](appendix-ii.html): [Expression statement](appendix-ii.html#expression-statement), [Print statement](appendix-ii.html#print-statement).

</aside>

Chạy script sinh AST và chiêm ngưỡng file “Stmt.java” kết quả với các class cây cú pháp mà ta cần cho expression và `print` statement. Đừng quên thêm file này vào project IDE hoặc makefile hay bất cứ công cụ build nào bạn dùng.

### Parse statement

Phương thức `parse()` của parser, vốn parse và trả về một expression duy nhất, chỉ là một mẹo tạm thời để chương trước chạy được. Giờ khi grammar của ta đã có quy tắc bắt đầu đúng là `program`, ta có thể biến `parse()` thành phiên bản “xịn”.

^code parse

<aside name="parse-error-handling">

Còn đoạn code bắt exception `ParseError` trước đây thì sao? Ta sẽ sớm bổ sung xử lý lỗi parse tốt hơn khi thêm hỗ trợ cho các loại statement khác.

</aside>

Hàm này parse một loạt statement, càng nhiều càng tốt cho đến khi gặp cuối input. Đây là bản dịch khá trực tiếp của quy tắc `program` sang phong cách đệ quy xuống (recursive descent). Ta cũng phải khấn một lời cầu nguyện nhỏ tới “các vị thần” Java verbosity vì giờ ta đang dùng ArrayList.

^code parser-imports (2 before, 1 after)

Một program là một danh sách statement, và ta parse từng statement bằng phương thức này:

^code parse-statement

Hơi sơ sài, nhưng ta sẽ bổ sung thêm các loại statement khác sau. Ta xác định quy tắc statement cụ thể nào được khớp bằng cách nhìn vào token hiện tại. Một token `print` thì rõ ràng là một `print` statement.

Nếu token tiếp theo không giống bất kỳ loại statement nào đã biết, ta giả định nó là một expression statement. Đây là trường hợp rơi xuống cuối cùng điển hình khi parse statement, vì khó mà nhận diện một expression chỉ từ token đầu tiên của nó.

Mỗi loại statement có phương thức riêng. Đầu tiên là `print`:

^code parse-print-statement

Vì ta đã khớp và tiêu thụ token `print` rồi, nên không cần làm lại ở đây. Ta parse expression tiếp theo, tiêu thụ dấu chấm phẩy kết thúc, và tạo ra cây cú pháp.

Nếu không khớp `print` statement, ta sẽ có một trong số này:

^code parse-expression-statement

Tương tự phương thức trước, ta parse một expression theo sau bởi dấu chấm phẩy. Ta bọc Expr đó trong một Stmt đúng loại và trả về.

### Execute statement

Ta đang đi lại những bước của vài chương trước ở dạng thu nhỏ, lần lượt qua phần front end. Parser của ta giờ có thể tạo ra cây cú pháp statement, nên bước tiếp theo và cuối cùng là thông dịch chúng. Giống như với expression, ta dùng Visitor pattern, nhưng ta có một interface visitor mới, `Stmt.Visitor`, để hiện thực vì statement có base class riêng.

Ta thêm interface đó vào danh sách các interface mà Interpreter implements.

^code interpreter (1 after)

<aside name="void">

Java không cho bạn dùng “void” viết thường làm đối số kiểu generic vì những lý do mơ hồ liên quan đến type erasure và stack. Thay vào đó, có một kiểu riêng “Void” dành cho mục đích này. Kiểu này giống như một “boxed void”, tương tự như “Integer” là cho “int”.

</aside>

Không giống expression, statement không tạo ra giá trị, nên kiểu trả về của các phương thức visit là Void, không phải Object. Ta có hai loại statement, và cần một phương thức visit cho mỗi loại. Dễ nhất là expression statement.

^code visit-expression-stmt

Ta đánh giá expression bên trong bằng phương thức `evaluate()` hiện có và <span name="discard">bỏ qua</span> giá trị trả về. Sau đó ta trả về `null`. Java yêu cầu điều đó để thỏa mãn kiểu trả về Void viết hoa đặc biệt. Kỳ lạ, nhưng biết làm sao được.

<aside name="discard">

Thật hợp lý, ta bỏ qua giá trị trả về từ `evaluate()` bằng cách đặt lời gọi đó bên trong một expression statement *của Java*.

</aside>

Phương thức visit cho `print` statement cũng không khác mấy.

^code visit-print

Trước khi bỏ qua giá trị của expression, ta chuyển nó thành chuỗi bằng phương thức `stringify()` mà ta đã giới thiệu ở chương trước, rồi in nó ra stdout.

Interpreter của ta giờ đã có thể “thăm” các statement, nhưng ta cần làm thêm chút việc để đưa chúng vào. Đầu tiên, sửa phương thức `interpret()` cũ trong class Interpreter để nhận một danh sách statement — nói cách khác là một chương trình.

^code interpret

Điều này thay thế đoạn code cũ vốn nhận một expression duy nhất. Code mới dựa vào helper nhỏ này:

^code execute

Đây là “phiên bản statement” của phương thức `evaluate()` mà ta có cho expression. Vì giờ ta làm việc với danh sách, ta cần cho Java biết điều đó.

^code import-list (2 before, 2 after)

Class Lox chính vẫn đang cố parse một expression duy nhất và truyền nó vào interpreter. Ta sửa dòng parse như sau:

^code parse-statements (1 before, 2 after)

Rồi thay lời gọi interpreter bằng:

^code interpret-statements (2 before, 1 after)

Về cơ bản chỉ là “đi đường ống” cho cú pháp mới. OK, chạy interpreter và thử xem. Lúc này, đáng để phác thảo một chương trình Lox nhỏ trong file văn bản để chạy như script. Ví dụ:

```lox
print "one";
print true;
print 2 + 1;
```

Trông gần giống một chương trình thực sự rồi! Lưu ý rằng REPL giờ cũng yêu cầu bạn nhập một statement đầy đủ thay vì chỉ một expression. Đừng quên dấu chấm phẩy.

## Biến toàn cục (Global Variables)

Giờ ta đã có statement, ta có thể bắt đầu làm việc với state. Trước khi đi vào toàn bộ sự phức tạp của lexical scoping, ta sẽ bắt đầu với loại biến dễ nhất — <span name="globals">biến toàn cục</span>. Ta cần hai cấu trúc mới.

1.  **Variable declaration** statement tạo ra một biến mới.

    ```lox
    var beverage = "espresso";
    ```

    Điều này tạo một binding mới, gắn một tên (ở đây là `"beverage"`) với một giá trị (ở đây là chuỗi `"espresso"`).

2.  Sau đó, một **variable expression** truy cập binding đó. Khi identifier `"beverage"` được dùng như một expression, nó sẽ tìm giá trị gắn với tên đó và trả về.

    ```lox
    print beverage; // "espresso".
    ```

Sau này, ta sẽ thêm gán giá trị và block scope, nhưng chừng đó là đủ để bắt đầu.

<aside name="globals">

Global state thường bị mang tiếng xấu. Đúng là nhiều global state — đặc biệt là state *mutable* — khiến việc bảo trì các chương trình lớn trở nên khó khăn. Về mặt kỹ thuật phần mềm, việc giảm thiểu sử dụng chúng là điều tốt.

Nhưng khi bạn đang ráp một ngôn ngữ lập trình đơn giản, hoặc thậm chí đang học ngôn ngữ đầu tiên, sự đơn giản “phẳng” của biến toàn cục lại hữu ích. Ngôn ngữ đầu tiên của tôi là BASIC và, dù cuối cùng tôi đã vượt qua nó, nhưng thật tuyệt khi tôi không phải đau đầu với quy tắc scope trước khi có thể khiến máy tính làm những điều thú vị.

</aside>

### Cú pháp biến

Như trước, ta sẽ triển khai từ trước ra sau, bắt đầu với cú pháp. Variable declaration là statement, nhưng chúng khác với các statement khác, và ta sẽ tách grammar của statement thành hai phần để xử lý chúng. Lý do là grammar giới hạn nơi một số loại statement được phép xuất hiện.

Các mệnh đề trong statement điều khiển luồng — như nhánh then và else của `if` hoặc thân của `while` — mỗi cái là một statement duy nhất. Nhưng statement đó không được phép là một statement khai báo tên. Ví dụ này OK:

```lox
if (monday) print "Ugh, already?";
```

Nhưng ví dụ này thì không:

```lox
if (monday) var beverage = "espresso";
```

Ta *có thể* cho phép trường hợp sau, nhưng nó gây rối. Scope của biến `beverage` đó là gì? Nó có tồn tại sau câu lệnh `if` không? Nếu có, giá trị của nó là gì vào những ngày khác Monday? Biến đó có tồn tại chút nào vào những ngày đó không?

Code như vậy thật kỳ quặc, nên C, Java và các “bạn bè” của chúng đều không cho phép. Cứ như thể có hai mức <span name="brace">“độ ưu tiên”</span> cho statement. Một số nơi mà statement được phép xuất hiện — như bên trong block hoặc ở top-level — cho phép mọi loại statement, bao gồm cả declaration. Những nơi khác chỉ cho phép các statement “ưu tiên cao” hơn, tức là không khai báo tên.

<aside name="brace">

Trong phép so sánh này, block statement hoạt động giống như dấu ngoặc đơn đối với expression. Bản thân block nằm ở “mức ưu tiên cao” và có thể dùng ở bất kỳ đâu, như trong các mệnh đề của `if`. Nhưng các statement *bên trong* nó có thể ở mức ưu tiên thấp hơn. Bạn được phép khai báo biến và các tên khác bên trong block. Dấu ngoặc nhọn cho phép bạn “thoát” trở lại grammar statement đầy đủ từ một nơi mà chỉ một số statement được phép.

</aside>


Để đáp ứng sự khác biệt này, ta thêm một quy tắc mới cho các loại statement khai báo tên.

```ebnf
program        → declaration* EOF ;

declaration    → varDecl
               | statement ;

statement      → exprStmt
               | printStmt ;
```

Các statement khai báo sẽ nằm dưới quy tắc `declaration` mới. Hiện tại, nó chỉ có biến, nhưng sau này sẽ bao gồm cả hàm và class. Bất kỳ nơi nào cho phép declaration cũng cho phép statement không khai báo, nên quy tắc `declaration` sẽ rơi xuống `statement`. Rõ ràng, bạn có thể khai báo ở top-level của script, nên `program` sẽ dẫn tới quy tắc mới này.

Quy tắc khai báo biến trông như sau:

```ebnf
varDecl        → "var" IDENTIFIER ( "=" expression )? ";" ;
```

Giống hầu hết các statement, nó bắt đầu bằng một keyword. Trong trường hợp này là `var`. Sau đó là một token identifier cho tên biến được khai báo, theo sau là một expression khởi tạo tùy chọn. Cuối cùng, ta “thắt nơ” bằng dấu chấm phẩy.

Để truy cập một biến, ta định nghĩa một loại primary expression mới:

```ebnf
primary        → "true" | "false" | "nil"
               | NUMBER | STRING
               | "(" expression ")"
               | IDENTIFIER ;
```

Mệnh đề `IDENTIFIER` này khớp với một token identifier duy nhất, được hiểu là tên của biến đang được truy cập.

Các quy tắc grammar mới này sẽ có cây cú pháp tương ứng. Trong AST generator, ta thêm một node <span name="var-stmt-ast">statement mới</span> cho khai báo biến.

^code var-stmt-ast (1 before, 1 after)

<aside name="var-stmt-ast">

Code được generated cho node mới này nằm ở [Phụ lục II](appendix-ii.html#variable-statement).

</aside>

Nó lưu token tên để biết đang khai báo gì, cùng với expression khởi tạo. (Nếu không có khởi tạo, trường này sẽ là `null`.)

Tiếp theo, ta thêm một node expression để truy cập biến.

^code var-expr (1 before, 1 after)

<span name="var-expr-ast">Nó</span> chỉ đơn giản là một lớp bọc quanh token tên biến. Hết. Như mọi khi, đừng quên chạy script sinh AST để có các file "Expr.java" và "Stmt.java" đã được cập nhật.

<aside name="var-expr-ast">

Code được generated cho node mới này nằm ở [Phụ lục II](appendix-ii.html#variable-expression).

</aside>

### Parse biến

Trước khi parse variable statement, ta cần sắp xếp lại một chút code để dành chỗ cho quy tắc `declaration` mới trong grammar. Top-level của một chương trình giờ là danh sách declaration, nên phương thức entrypoint của parser sẽ thay đổi.

^code parse-declaration (3 before, 4 after)

Nó gọi tới phương thức mới này:

^code declaration

Này, bạn còn nhớ ở [chương trước](parsing-expressions.html) khi ta đã chuẩn bị hạ tầng để xử lý khôi phục lỗi không? Giờ ta cuối cùng cũng sẵn sàng kết nối nó.

Phương thức `declaration()` này là phương thức được gọi lặp lại khi parse một loạt statement trong block hoặc script, nên đây là nơi thích hợp để đồng bộ khi parser vào chế độ panic. Toàn bộ thân hàm được bọc trong một khối try để bắt exception được ném ra khi parser bắt đầu khôi phục lỗi. Điều này giúp nó quay lại thử parse phần bắt đầu của statement hoặc declaration tiếp theo.

Việc parse thực sự diễn ra bên trong khối try. Đầu tiên, nó kiểm tra xem ta có đang ở một variable declaration không bằng cách tìm keyword `var` ở đầu. Nếu không, nó rơi xuống phương thức `statement()` hiện có để parse `print` và expression statement.

Nhớ rằng `statement()` sẽ cố parse một expression statement nếu không khớp loại statement nào khác? Và `expression()` sẽ báo lỗi cú pháp nếu không thể parse một expression tại token hiện tại? Chuỗi lời gọi này đảm bảo ta báo lỗi nếu không parse được một declaration hoặc statement hợp lệ.

Khi parser khớp token `var`, nó sẽ rẽ sang:

^code parse-var-declaration

Như thường lệ, code recursive descent sẽ bám sát quy tắc grammar. Parser đã khớp token `var`, nên tiếp theo nó yêu cầu và tiêu thụ một token identifier cho tên biến.

Sau đó, nếu thấy token `=`, nó biết có một expression khởi tạo và sẽ parse nó. Nếu không, nó để initializer là `null`. Cuối cùng, nó tiêu thụ dấu chấm phẩy bắt buộc ở cuối statement. Tất cả được gói trong một node cú pháp Stmt.Var và ta xong phần này.

Parse một variable expression còn dễ hơn. Trong `primary()`, ta tìm token identifier.

^code parse-identifier (2 before, 2 after)

Vậy là ta đã có một front-end hoạt động để khai báo và sử dụng biến. Việc còn lại là đưa nó vào interpreter. Trước khi làm điều đó, ta cần nói về việc biến “sống” ở đâu trong bộ nhớ.


## Environment

Các binding gắn biến với giá trị của chúng cần được lưu trữ ở đâu đó.  
Từ khi những người làm Lisp phát minh ra dấu ngoặc, cấu trúc dữ liệu này đã được gọi là <span name="env">**environment**</span>.

<img src="image/statements-and-state/environment.png" alt="Một environment chứa hai binding." />

<aside name="env">

Tôi thích tưởng tượng environment theo nghĩa đen, như một khu rừng yên bình nơi các biến và giá trị vui đùa cùng nhau.

</aside>

Bạn có thể hình dung nó như một <span name="map">map</span> mà key là tên biến và value là… giá trị của biến. Thực tế, đó chính là cách ta sẽ hiện thực nó trong Java. Ta có thể nhét map này và code quản lý nó trực tiếp vào Interpreter, nhưng vì nó là một khái niệm tách biệt rõ ràng, ta sẽ tách nó thành một class riêng.

Tạo một file mới và thêm:

<aside name="map">

Java gọi chúng là **map** hoặc **hashmap**. Các ngôn ngữ khác gọi là **hash table**, **dictionary** (Python và C#), **hash** (Ruby và Perl), **table** (Lua), hoặc **associative array** (PHP). Ngày xưa, chúng còn được gọi là **scatter table**.

</aside>

^code environment-class

Bên trong có một Java Map để lưu các binding. Nó dùng chuỗi thuần làm key, không dùng token. Một token đại diện cho một đơn vị code tại một vị trí cụ thể trong mã nguồn, nhưng khi tra cứu biến, tất cả token identifier có cùng tên nên tham chiếu tới cùng một biến (tạm bỏ qua scope). Dùng chuỗi thuần đảm bảo tất cả các token đó trỏ tới cùng một key trong map.

Có hai thao tác ta cần hỗ trợ. Đầu tiên, khai báo biến sẽ bind một tên mới với một giá trị.

^code environment-define

Không có gì phức tạp, nhưng ta đã đưa ra một lựa chọn ngữ nghĩa thú vị. Khi thêm key vào map, ta không kiểm tra xem nó đã tồn tại chưa. Điều đó nghĩa là chương trình này chạy được:

```lox
var a = "before";
print a; // "before".
var a = "after";
print a; // "after".
```

Một variable statement không chỉ định nghĩa một biến *mới*, nó cũng có thể được dùng để *định nghĩa lại* một biến đã tồn tại. Ta <span name="scheme">có thể</span> chọn coi đây là lỗi. Người dùng có thể không định ghi đè biến đã có. (Nếu họ muốn, có lẽ họ sẽ dùng phép gán, không phải `var`.) Việc coi redefinition là lỗi sẽ giúp họ phát hiện bug đó.

Tuy nhiên, làm vậy lại không hợp với REPL. Trong một phiên REPL, thật tiện khi không phải nhớ mình đã khai báo biến nào. Ta có thể cho phép redefinition trong REPL nhưng không cho trong script, nhưng như vậy người dùng sẽ phải học hai bộ quy tắc, và code copy-paste từ dạng này sang dạng kia có thể không chạy được.

<aside name="scheme">

Quy tắc của tôi về biến và scope là: “Khi phân vân, hãy làm như Scheme”. Những người làm Scheme có lẽ đã dành nhiều thời gian suy nghĩ về scope hơn chúng ta — một trong những mục tiêu chính của Scheme là giới thiệu lexical scoping cho thế giới — nên đi theo họ là lựa chọn an toàn.

Scheme cho phép định nghĩa lại biến ở top-level.

</aside>

Vì vậy, để giữ cho hai chế độ nhất quán, ta sẽ cho phép điều đó — ít nhất là với biến toàn cục. Khi một biến đã tồn tại, ta cần cách để tra cứu nó.

^code environment-get (2 before, 1 after)

Phần này thú vị hơn một chút về mặt ngữ nghĩa. Nếu tìm thấy biến, nó đơn giản trả về giá trị đã bind. Nhưng nếu không tìm thấy thì sao? Một lần nữa, ta có vài lựa chọn:

* Coi đó là lỗi cú pháp.

* Coi đó là lỗi runtime.

* Cho phép và trả về một giá trị mặc định như `nil`.

Lox khá “thoáng”, nhưng lựa chọn cuối có vẻ *quá* dễ dãi. Coi đó là lỗi cú pháp — lỗi ở compile-time — có vẻ hợp lý. Dùng một biến chưa được định nghĩa là bug, và phát hiện càng sớm càng tốt.

Vấn đề là *dùng* một biến không giống với *tham chiếu* tới nó. Bạn có thể tham chiếu tới một biến trong một đoạn code mà không đánh giá nó ngay nếu đoạn code đó nằm trong một hàm. Nếu ta coi việc *nhắc tới* biến trước khi khai báo là lỗi tĩnh, thì sẽ khó hơn nhiều để định nghĩa hàm đệ quy.

Ta có thể xử lý được đệ quy đơn — một hàm tự gọi chính nó — bằng cách khai báo tên hàm trước khi phân tích thân hàm. Nhưng điều đó không giúp gì cho các thủ tục đệ quy tương hỗ gọi lẫn nhau. Xem ví dụ:


```lox
fun isOdd(n) {
  if (n == 0) return false;
  return isEven(n - 1);
}

fun isEven(n) {
  if (n == 0) return true;
  return isOdd(n - 1);
}
```

<aside name="contrived">

Đúng là đây có lẽ không phải cách hiệu quả nhất để kiểm tra một số là chẵn hay lẻ (chưa kể những điều tệ hại xảy ra nếu bạn truyền vào một số không nguyên hoặc số âm). Nhưng hãy tạm chấp nhận ví dụ này nhé.

</aside>

Hàm `isEven()` chưa được định nghĩa tại <span name="declare">thời điểm</span> ta đang xem phần thân của `isOdd()` nơi nó được gọi. Nếu ta đảo thứ tự hai hàm này, thì `isOdd()` lại chưa được định nghĩa khi ta đang xem phần thân của `isEven()`.

<aside name="declare">

Một số ngôn ngữ kiểu tĩnh như Java và C# giải quyết vấn đề này bằng cách quy định rằng top-level của một chương trình không phải là một chuỗi các câu lệnh imperative. Thay vào đó, chương trình là một tập hợp các khai báo, tất cả cùng “ra đời” đồng thời. Trình biên dịch sẽ khai báo *tất cả* các tên trước khi xem phần thân của *bất kỳ* hàm nào.

Các ngôn ngữ cũ hơn như C và Pascal thì không hoạt động như vậy. Thay vào đó, chúng buộc bạn phải thêm *forward declaration* tường minh để khai báo tên trước khi nó được định nghĩa đầy đủ. Đây là sự nhượng bộ cho giới hạn sức mạnh tính toán thời đó. Họ muốn có thể biên dịch một file nguồn chỉ trong một lần duyệt qua văn bản, nên các compiler đó không thể gom tất cả khai báo trước khi xử lý phần thân hàm.

</aside>

Vì việc coi đây là lỗi *tĩnh* sẽ khiến khai báo đệ quy trở nên quá khó khăn, ta sẽ hoãn lỗi này sang runtime. Việc tham chiếu tới một biến trước khi nó được định nghĩa là chấp nhận được miễn là bạn không *đánh giá* tham chiếu đó. Điều này cho phép chương trình kiểm tra số chẵn/lẻ hoạt động, nhưng bạn sẽ gặp lỗi runtime trong trường hợp:

```lox
print a;
var a = "too late!";
```

Giống như với lỗi kiểu dữ liệu trong phần code đánh giá expression, ta báo lỗi runtime bằng cách ném ra một exception. Exception này chứa token của biến để ta có thể cho người dùng biết chính xác vị trí trong code mà họ gặp lỗi.

### Thông dịch biến toàn cục

Class Interpreter sẽ có một instance của class Environment mới.

^code environment-field (2 before, 1 after)

Ta lưu nó như một field trực tiếp trong Interpreter để các biến tồn tại trong bộ nhớ miễn là interpreter còn chạy.

Ta có hai cây cú pháp mới, nên sẽ có hai phương thức visit mới. Đầu tiên là cho statement khai báo biến.

^code visit-var

Nếu biến có initializer, ta sẽ đánh giá nó. Nếu không, ta lại có một lựa chọn khác. Ta có thể biến điều này thành lỗi cú pháp trong parser bằng cách *bắt buộc* phải có initializer. Tuy nhiên, hầu hết các ngôn ngữ không làm vậy, nên áp dụng điều đó cho Lox có vẻ hơi khắt khe.

Ta cũng có thể biến nó thành lỗi runtime. Nghĩa là cho phép bạn định nghĩa một biến chưa khởi tạo, nhưng nếu truy cập nó trước khi gán giá trị, sẽ xảy ra lỗi runtime. Đây không phải ý tưởng tồi, nhưng hầu hết các ngôn ngữ kiểu động cũng không làm vậy. Thay vào đó, ta sẽ giữ mọi thứ đơn giản và quy định rằng Lox sẽ gán `nil` cho biến nếu nó không được khởi tạo tường minh.

```lox
var a;
print a; // "nil".
```

Vì vậy, nếu không có initializer, ta gán giá trị `null`, tức là cách Java biểu diễn giá trị `nil` của Lox. Sau đó, ta yêu cầu environment bind biến đó với giá trị này.

Tiếp theo, ta đánh giá một variable expression.

^code visit-variable

Phần này chỉ đơn giản là chuyển tiếp sang environment, nơi thực hiện công việc chính để đảm bảo biến đã được định nghĩa. Với điều đó, ta đã có biến hoạt động ở mức cơ bản. Hãy thử:

```lox
var a = 1;
var b = 2;
print a + b;
```

Ta chưa thể tái sử dụng *code*, nhưng đã có thể bắt đầu xây dựng chương trình tái sử dụng *dữ liệu*.

## Gán giá trị (Assignment)

Hoàn toàn có thể tạo ra một ngôn ngữ có biến nhưng không cho phép gán lại — hay **mutate** — chúng. Haskell là một ví dụ. SML chỉ hỗ trợ tham chiếu và mảng mutable — biến không thể gán lại. Rust thì “hướng” bạn tránh mutation bằng cách yêu cầu từ khóa `mut` để bật tính năng gán.

Việc mutate một biến là một side effect và, như tên gọi, một số người trong giới ngôn ngữ cho rằng side effect là thứ <span name="pure">“bẩn”</span> hoặc thiếu tinh tế. Code nên là toán học thuần khiết tạo ra các giá trị — trong suốt, bất biến — như một hành động sáng tạo thần thánh. Không phải một cỗ máy cục mịch đập nặn dữ liệu thành hình, từng cú lệnh imperative một.

<aside name="pure">

Tôi thấy thú vị ở chỗ chính nhóm người tự hào về tư duy logic lạnh lùng lại là những người không cưỡng được việc dùng các thuật ngữ đầy cảm xúc cho công việc của mình: “pure”, “side effect”, “lazy”, “persistent”, “first-class”, “higher-order”.

</aside>

Lox thì không khắc khổ như vậy. Lox là một ngôn ngữ imperative, và mutation là điều hiển nhiên. Việc thêm hỗ trợ cho assignment không đòi hỏi nhiều công sức. Biến toàn cục vốn đã hỗ trợ định nghĩa lại, nên hầu hết cơ chế đã sẵn sàng. Chủ yếu, ta chỉ còn thiếu cú pháp gán tường minh.

### Cú pháp gán (Assignment syntax)

Cú pháp nhỏ bé `=` này phức tạp hơn bạn tưởng. Giống như hầu hết các ngôn ngữ bắt nguồn từ C, phép gán là một <span name="assign">expression</span> chứ không phải statement. Như trong C, nó là dạng expression có độ ưu tiên thấp nhất. Điều đó có nghĩa là quy tắc của nó nằm giữa `expression` và `equality` (dạng expression có độ ưu tiên thấp kế tiếp).

<aside name="assign">

Trong một số ngôn ngữ khác, như Pascal, Python và Go, phép gán là một statement.

</aside>

```ebnf
expression     → assignment ;
assignment     → IDENTIFIER "=" assignment
               | equality ;
```

Điều này nói rằng một `assignment` hoặc là một identifier theo sau bởi dấu `=` và một expression cho giá trị, hoặc là một `equality` (và do đó là bất kỳ expression nào khác). Sau này, `assignment` sẽ phức tạp hơn khi ta thêm setter cho thuộc tính của object, như:

```lox
instance.field = "value";
```

Phần dễ là thêm <span name="assign-ast">node cây cú pháp mới</span>.

^code assign-expr (1 before, 1 after)

<aside name="assign-ast">

Code được generated cho node mới này nằm ở [Phụ lục II](appendix-ii.html#assign-expression).

</aside>

Node này có một token cho biến được gán và một expression cho giá trị mới. Sau khi bạn chạy AstGenerator để có class `Expr.Assign` mới, hãy thay phần thân của phương thức `expression()` trong parser để khớp với quy tắc đã cập nhật.

^code expression (1 before, 1 after)

Đây là phần bắt đầu trở nên phức tạp. Một parser recursive descent với lookahead một token không thể nhìn đủ xa để biết rằng nó đang parse một phép gán cho đến *sau khi* nó đã đi qua phần bên trái và bắt gặp dấu `=`. Bạn có thể tự hỏi tại sao nó lại cần biết điều đó. Xét cho cùng, ta cũng không biết mình đang parse một biểu thức `+` cho đến khi parse xong toán hạng bên trái.

Sự khác biệt là phần bên trái của phép gán không phải là một expression trả về giá trị. Nó là một dạng “pseudo-expression” trả về một “thứ” mà bạn có thể gán vào. Xem ví dụ:

```lox
var a = "before";
a = "value";
```

Ở dòng thứ hai, ta không *đánh giá* `a` (vì như vậy sẽ trả về chuỗi `"before"`). Ta xác định biến `a` tham chiếu tới đâu để biết nơi lưu giá trị của expression bên phải. Các [thuật ngữ kinh điển](https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue) cho hai <span name="l-value">cấu trúc</span> này là **l-value** và **r-value**. Tất cả các expression mà ta đã thấy cho tới giờ, vốn tạo ra giá trị, đều là r-value. Một l-value “đánh giá” ra một vị trí lưu trữ mà bạn có thể gán giá trị vào đó.

<aside name="l-value">

Thực tế, các tên gọi này xuất phát từ phép gán: l-value xuất hiện ở *bên trái* dấu `=` trong phép gán, và r-value ở *bên phải*.

</aside>

Ta muốn cây cú pháp phản ánh rằng một l-value không được đánh giá như một expression bình thường. Đó là lý do node `Expr.Assign` có một *Token* cho phần bên trái, không phải một `Expr`. Vấn đề là parser không biết nó đang parse một l-value cho đến khi gặp dấu `=`. Trong một l-value phức tạp, điều đó có thể xảy ra <span name="many">nhiều</span> token sau.

```lox
makeList().head.next = node;
```

<aside name="many">

Vì phần nhận của một phép gán thuộc tính có thể là bất kỳ expression nào, và expression có thể dài tùy ý, nên có thể cần một số lượng token lookahead *không giới hạn* để tìm được dấu `=`.

</aside>

Ta chỉ có một token lookahead, vậy phải làm sao? Ta dùng một mẹo nhỏ, trông như thế này:

^code parse-assignment

Hầu hết code để parse một assignment expression trông giống với các toán tử nhị phân khác như `+`. Ta parse phần bên trái, vốn có thể là bất kỳ expression nào có độ ưu tiên cao hơn. Nếu tìm thấy dấu `=`, ta parse phần bên phải rồi gói tất cả lại trong một node cây cú pháp assignment.


<aside name="no-throw">

Ta *báo* lỗi nếu phía bên trái không phải là một mục tiêu gán hợp lệ, nhưng ta không *ném* lỗi vì parser không ở trạng thái rối loạn cần phải vào chế độ panic và đồng bộ lại.

</aside>

Một điểm khác biệt nhỏ so với các toán tử nhị phân là ta không lặp để xây dựng một chuỗi các toán tử giống nhau. Vì phép gán là right-associative, ta sẽ đệ quy gọi `assignment()` để parse phía bên phải.

Mẹo ở đây là ngay trước khi tạo node assignment expression, ta nhìn vào expression phía bên trái và xác định nó là loại mục tiêu gán nào. Ta chuyển đổi node expression r-value thành một biểu diễn l-value.

Việc chuyển đổi này hoạt động vì mọi mục tiêu gán hợp lệ tình cờ cũng là <span name="converse">cú pháp hợp lệ</span> của một expression bình thường. Xem ví dụ một phép gán field phức tạp như:

<aside name="converse">

Bạn vẫn có thể dùng mẹo này ngay cả khi có những mục tiêu gán không phải là expression hợp lệ. Hãy định nghĩa một **cover grammar**, một grammar “nới lỏng” chấp nhận tất cả cú pháp của expression *và* mục tiêu gán hợp lệ. Khi gặp dấu `=`, báo lỗi nếu phía bên trái không nằm trong grammar mục tiêu gán hợp lệ. Ngược lại, nếu *không* gặp dấu `=`, báo lỗi nếu phía bên trái không phải là một *expression* hợp lệ.

</aside>

```lox
newPoint(x + 2, 0).y = 3;
```

Phía bên trái của phép gán này cũng có thể là một expression hợp lệ:

```lox
newPoint(x + 2, 0).y;
```

Ví dụ đầu tiên gán giá trị cho field, ví dụ thứ hai lấy giá trị của field.

Điều này có nghĩa là ta có thể parse phía bên trái *như thể nó là* một expression và sau đó tạo cây cú pháp biến nó thành mục tiêu gán. Nếu expression phía bên trái không phải là một mục tiêu gán <span name="paren">hợp lệ</span>, ta sẽ báo lỗi cú pháp. Điều này đảm bảo ta báo lỗi cho code như:

```lox
a + b = c;
```

<aside name="paren">

Ở chương parse trước đây, tôi đã nói rằng ta biểu diễn expression trong ngoặc trong cây cú pháp vì sau này sẽ cần tới chúng. Đây chính là lý do. Ta cần phân biệt được các trường hợp:

```lox
a = 3;   // OK.
(a) = 3; // Lỗi.
```

</aside>

Hiện tại, mục tiêu gán hợp lệ duy nhất là một variable expression đơn giản, nhưng sau này ta sẽ thêm field. Kết quả cuối cùng của mẹo này là một node assignment expression biết nó đang gán cho cái gì và có một cây con expression cho giá trị được gán. Tất cả chỉ với một token lookahead và không cần backtracking.

### Ngữ nghĩa của phép gán

Ta có một node cây cú pháp mới, nên interpreter sẽ có một phương thức visit mới.

^code visit-assign

Vì lý do hiển nhiên, nó giống với khai báo biến. Nó đánh giá phía bên phải để lấy giá trị, rồi lưu giá trị đó vào biến được đặt tên. Thay vì dùng `define()` trên Environment, nó gọi phương thức mới này:

^code environment-assign

Điểm khác biệt chính giữa gán và khai báo là gán *không được* <span name="new">phép</span> tạo một biến *mới*. Trong hiện thực của ta, điều đó có nghĩa là sẽ xảy ra lỗi runtime nếu key chưa tồn tại trong map biến của environment.

<aside name="new">

Không giống Python và Ruby, Lox không có [implicit variable declaration](#design-note).

</aside>

Điều cuối cùng mà phương thức `visit()` làm là trả về giá trị vừa gán. Đó là vì phép gán là một expression có thể được lồng bên trong các expression khác, như:

```lox
var a = 1;
print a = 2; // "2".
```

Interpreter của ta giờ có thể tạo, đọc và sửa đổi biến. Nó gần như tinh vi ngang với các <span name="basic">BASIC</span> đời đầu. Biến toàn cục thì đơn giản, nhưng viết một chương trình lớn khi bất kỳ hai đoạn code nào cũng có thể vô tình ghi đè state của nhau thì chẳng vui chút nào. Ta muốn có biến *cục bộ*, nghĩa là đã đến lúc nói về *scope*.

<aside name="basic">

Có lẽ còn tốt hơn một chút. Không giống một số BASIC cũ, Lox có thể xử lý tên biến dài hơn hai ký tự.

</aside>


## Scope (Phạm vi)

**Scope** (phạm vi) định nghĩa một vùng mà trong đó một tên được ánh xạ tới một thực thể nhất định. Nhiều scope cho phép cùng một tên có thể tham chiếu tới những thứ khác nhau trong các ngữ cảnh khác nhau. Ở nhà tôi, “Bob” thường chỉ tôi. Nhưng có thể ở thị trấn của bạn, bạn lại biết một Bob khác. Cùng tên, nhưng là những người khác nhau tùy vào nơi bạn nhắc tới.

<span name="lexical">**Lexical scope**</span> (hay ít phổ biến hơn là **static scope**) là một kiểu scoping cụ thể, trong đó chính văn bản của chương trình cho thấy phạm vi bắt đầu và kết thúc ở đâu. Trong Lox, cũng như hầu hết các ngôn ngữ hiện đại, biến được scope theo kiểu lexical. Khi bạn thấy một expression sử dụng một biến nào đó, bạn có thể xác định nó tham chiếu tới khai báo biến nào chỉ bằng cách đọc code tĩnh, không cần chạy chương trình.

<aside name="lexical">

“Lexical” bắt nguồn từ tiếng Hy Lạp “lexikos” nghĩa là “liên quan tới từ ngữ”. Khi dùng trong ngôn ngữ lập trình, nó thường có nghĩa là một điều bạn có thể xác định từ chính mã nguồn mà không cần execute.

Lexical scope xuất hiện cùng ALGOL. Các ngôn ngữ trước đó thường dùng dynamic scope. Các nhà khoa học máy tính khi đó tin rằng dynamic scope chạy nhanh hơn. Ngày nay, nhờ những hacker Scheme thời kỳ đầu, ta biết điều đó không đúng. Thậm chí, thực tế còn ngược lại.

Dynamic scope cho biến vẫn tồn tại ở một vài nơi. Emacs Lisp mặc định dùng dynamic scope cho biến. Macro [`binding`](http://clojuredocs.org/clojure.core/binding) trong Clojure cung cấp nó. Câu lệnh [`with`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with) vốn bị nhiều người chê trong JavaScript biến các thuộc tính của một object thành các biến có dynamic scope.

</aside>

Ví dụ:

```lox
{
  var a = "first";
  print a; // "first".
}

{
  var a = "second";
  print a; // "second".
}
```

Ở đây, ta có hai block, mỗi block khai báo một biến `a`. Bạn và tôi chỉ cần nhìn code là biết rằng `a` trong câu lệnh `print` đầu tiên tham chiếu tới biến `a` đầu tiên, và `a` trong câu lệnh thứ hai tham chiếu tới biến `a` thứ hai.

<img src="image/statements-and-state/blocks.png" alt="Một environment cho mỗi biến 'a'." />

Điều này trái ngược với **dynamic scope**, nơi bạn không biết một tên tham chiếu tới cái gì cho tới khi chạy code. Lox không có biến với dynamic scope, nhưng method và field trên object thì có dynamic scope.

```lox
class Saxophone {
  play() {
    print "Careless Whisper";
  }
}

class GolfClub {
  play() {
    print "Fore!";
  }
}

fun playIt(thing) {
  thing.play();
}
```

Khi `playIt()` gọi `thing.play()`, ta không biết sẽ nghe “Careless Whisper” hay “Fore!”. Điều đó phụ thuộc vào việc bạn truyền một Saxophone hay một GolfClub vào hàm, và ta chỉ biết điều đó khi runtime.

Scope và environment là hai khái niệm gần gũi. Scope là khái niệm lý thuyết, còn environment là cơ chế hiện thực nó. Khi interpreter chạy qua code, các node trong cây cú pháp ảnh hưởng tới scope sẽ thay đổi environment. Trong cú pháp kiểu C như của Lox, scope được điều khiển bởi các block trong ngoặc nhọn. (Đó là lý do ta gọi nó là **block scope**.)

```lox
{
  var a = "in block";
}
print a; // Lỗi! Không còn "a".
```

Bắt đầu một block sẽ tạo ra một local scope mới, và scope đó kết thúc khi execute qua dấu `}` đóng. Bất kỳ biến nào khai báo bên trong block sẽ biến mất.

### Lồng nhau & che khuất (Nesting and shadowing)

Cách tiếp cận đầu tiên để hiện thực block scope có thể như sau:

1.  Khi duyệt từng statement bên trong block, theo dõi bất kỳ biến nào được khai báo.
2.  Sau khi statement cuối cùng được execute, yêu cầu environment xóa tất cả các biến đó.

Cách này sẽ hoạt động với ví dụ trước. Nhưng hãy nhớ, một trong những lý do để có local scope là để đóng gói (encapsulation) — một khối code ở một góc của chương trình không nên can thiệp vào một khối khác. Xem ví dụ sau:


```lox
// How loud?
var volume = 11;

// Silence.
volume = 0;

// Calculate size of 3x4x5 cuboid.
{
  var volume = 3 * 4 * 5;
  print volume;
}
```

Hãy xem block nơi ta tính thể tích của hình hộp chữ nhật bằng cách khai báo cục bộ biến `volume`. Sau khi thoát khỏi block, interpreter sẽ xóa biến `volume` *toàn cục*. Điều đó là không đúng. Khi thoát khỏi block, ta chỉ nên xóa các biến được khai báo bên trong block, nhưng nếu có một biến cùng tên được khai báo bên ngoài block, *đó là một biến khác*. Nó không nên bị đụng tới.

Khi một biến cục bộ có cùng tên với một biến trong scope bao ngoài, nó sẽ **che khuất** (shadow) biến bên ngoài đó. Code bên trong block sẽ không thể thấy biến bên ngoài nữa — nó bị “che” bởi biến bên trong — nhưng biến bên ngoài vẫn tồn tại.

Khi ta bước vào một block scope mới, ta cần giữ nguyên các biến được định nghĩa ở scope bên ngoài để chúng vẫn tồn tại khi ta thoát khỏi block bên trong. Ta làm điều đó bằng cách tạo một environment mới cho mỗi block, chỉ chứa các biến được định nghĩa trong scope đó. Khi thoát khỏi block, ta loại bỏ environment của nó và khôi phục environment trước đó.

Ta cũng cần xử lý các biến ở scope bao ngoài *không* bị shadow.

```lox
var global = "outside";
{
  var local = "inside";
  print global + local;
}
```

Ở đây, `global` nằm trong environment toàn cục bên ngoài và `local` được định nghĩa bên trong environment của block. Trong câu lệnh `print` đó, cả hai biến đều nằm trong scope. Để tìm chúng, interpreter phải tìm kiếm không chỉ trong environment trong cùng hiện tại, mà còn ở tất cả các environment bao ngoài.

Ta hiện thực điều này bằng cách <span name="cactus">xâu chuỗi</span> các environment lại với nhau. Mỗi environment có một tham chiếu tới environment của scope bao ngoài trực tiếp. Khi tra cứu một biến, ta đi dọc chuỗi này từ trong ra ngoài cho tới khi tìm thấy biến. Bắt đầu từ scope trong cùng là cách ta làm cho biến cục bộ che khuất biến bên ngoài.

<img src="image/statements-and-state/chaining.png" alt="Các environment cho mỗi scope, được liên kết với nhau." />

<aside name="cactus">

Khi interpreter đang chạy, các environment tạo thành một danh sách tuyến tính các object, nhưng nếu xét toàn bộ tập hợp environment được tạo ra trong suốt quá trình execute, một scope bên ngoài có thể chứa nhiều block lồng bên trong, và mỗi block sẽ trỏ tới scope bên ngoài đó, tạo thành một cấu trúc dạng cây, dù tại một thời điểm chỉ tồn tại một đường đi qua cây.

Tên gọi khô khan cho cấu trúc này là [**parent-pointer tree**](https://en.wikipedia.org/wiki/Parent_pointer_tree), nhưng tôi thích cái tên gợi hình hơn: **cactus stack**.

<img class="above" src="image/statements-and-state/cactus.png" alt="Mỗi nhánh trỏ tới cha của nó. Gốc là scope toàn cục." />

</aside>

Trước khi thêm cú pháp block vào grammar, ta sẽ nâng cấp class Environment để hỗ trợ việc lồng nhau này. Đầu tiên, ta cho mỗi environment một tham chiếu tới environment bao ngoài của nó.

^code enclosing-field (1 before, 1 after)

Trường này cần được khởi tạo, nên ta thêm một vài constructor.

^code environment-constructors

Constructor không tham số dành cho environment của scope toàn cục, nơi kết thúc chuỗi. Constructor còn lại tạo một scope cục bộ mới lồng bên trong scope ngoài được truyền vào.

Ta không cần đụng tới phương thức `define()` — một biến mới luôn được khai báo trong scope trong cùng hiện tại. Nhưng việc tra cứu và gán giá trị cho biến làm việc với các biến đã tồn tại, và chúng cần đi dọc chuỗi để tìm. Đầu tiên là tra cứu:

^code environment-get-enclosing (2 before, 3 after)

Nếu biến không được tìm thấy trong environment này, ta đơn giản thử ở environment bao ngoài. Environment đó lại làm điều tương tự <span name="recurse">đệ quy</span>, nên cuối cùng sẽ duyệt qua toàn bộ chuỗi. Nếu ta tới một environment không có bao ngoài và vẫn không tìm thấy biến, thì ta bỏ cuộc và báo lỗi như trước.

Việc gán giá trị hoạt động tương tự.

<aside name="recurse">

Có thể việc duyệt chuỗi theo cách lặp sẽ nhanh hơn, nhưng tôi nghĩ giải pháp đệ quy trông đẹp hơn. Trong clox, ta sẽ làm một cách *nhanh hơn nhiều*.

</aside>

^code environment-assign-enclosing (4 before, 1 after)

Một lần nữa, nếu biến không có trong environment này, nó sẽ kiểm tra ở scope ngoài, đệ quy.

### Cú pháp & ngữ nghĩa của Block

Giờ khi Environment đã có thể lồng nhau, ta sẵn sàng thêm block vào ngôn ngữ. Đây là grammar:

```ebnf
statement      → exprStmt
               | printStmt
               | block ;

block          → "{" declaration* "}" ;
```

Một block là một chuỗi (có thể rỗng) các statement hoặc declaration được bao quanh bởi dấu ngoặc nhọn. Bản thân block là một statement và có thể xuất hiện ở bất kỳ nơi nào statement được phép. Node <span name="block-ast">cây cú pháp</span> trông như sau:

^code block-ast (1 before, 1 after)

<aside name="block-ast">

Code được generated cho node mới này nằm ở [Phụ lục II](appendix-ii.html#block-statement).

</aside>

<span name="generate">Nó</span> chứa danh sách các statement nằm bên trong block. Việc parse khá đơn giản. Giống như các statement khác, ta nhận diện phần bắt đầu của block bằng token mở đầu — trong trường hợp này là `{`. Trong phương thức `statement()`, ta thêm:

<aside name="generate">

Như mọi khi, đừng quên chạy "GenerateAst.java".

</aside>

^code parse-block (1 before, 2 after)

Toàn bộ công việc chính diễn ra ở đây:

^code block

Ta <span name="list">tạo</span> một danh sách rỗng, sau đó parse các statement và thêm chúng vào danh sách cho đến khi gặp dấu `}` đóng, đánh dấu kết thúc block. Lưu ý rằng vòng lặp cũng có kiểm tra `isAtEnd()` một cách tường minh. Ta phải cẩn thận để tránh vòng lặp vô hạn, ngay cả khi đang parse code không hợp lệ. Nếu người dùng quên dấu `}` đóng, parser cần đảm bảo không bị kẹt.

<aside name="list">

Việc để `block()` trả về danh sách statement thô và để `statement()` bọc danh sách đó trong một Stmt.Block trông hơi lạ. Tôi làm vậy vì sau này ta sẽ tái sử dụng `block()` để parse thân hàm và ta không muốn thân hàm đó bị bọc trong một Stmt.Block.

</aside>

Vậy là xong phần cú pháp. Về ngữ nghĩa, ta thêm một phương thức visit mới vào Interpreter.

^code visit-block

Để execute một block, ta tạo một environment mới cho scope của block và chuyển nó sang phương thức khác này:

^code execute-block

Phương thức mới này execute một danh sách statement trong ngữ cảnh của một <span name="param">environment</span> được truyền vào. Cho tới giờ, trường `environment` trong Interpreter luôn trỏ tới cùng một environment — environment toàn cục. Giờ đây, trường này đại diện cho environment *hiện tại*, tức là environment tương ứng với scope trong cùng chứa code đang được execute.

Để execute code trong một scope nhất định, phương thức này cập nhật trường `environment` của interpreter, duyệt qua tất cả statement, rồi khôi phục giá trị trước đó. Như một thói quen tốt trong Java, nó khôi phục environment trước đó bằng một khối finally. Nhờ vậy, environment sẽ được khôi phục ngay cả khi có exception xảy ra.

<aside name="param">

Việc thay đổi và khôi phục thủ công một trường `environment` mutable có cảm giác hơi thiếu tinh tế. Một cách tiếp cận kinh điển khác là truyền tường minh environment như một tham số cho mỗi phương thức visit. Để “thay đổi” environment, bạn truyền một environment khác khi đệ quy xuống cây. Bạn không cần khôi phục cái cũ, vì environment mới nằm trên Java stack và sẽ tự động bị loại bỏ khi interpreter thoát khỏi phương thức visit của block.

Tôi đã cân nhắc cách này cho jlox, nhưng việc thêm tham số environment vào từng phương thức visit khá tẻ nhạt và dài dòng. Để cuốn sách đơn giản hơn một chút, tôi chọn cách dùng trường mutable.

</aside>

Thật bất ngờ, đó là tất cả những gì ta cần làm để hỗ trợ đầy đủ biến cục bộ, lồng nhau và shadowing. Thử xem:

```lox
var a = "global a";
var b = "global b";
var c = "global c";
{
  var a = "outer a";
  var b = "outer b";
  {
    var a = "inner a";
    print a;
    print b;
    print c;
  }
  print a;
  print b;
  print c;
}
print a;
print b;
print c;
```

Interpreter nhỏ bé của chúng ta giờ đã có thể “ghi nhớ” mọi thứ. Ta đang tiến gần hơn tới một thứ giống như một ngôn ngữ lập trình đầy đủ tính năng.

<div class="challenges">

## Thử thách

1.  REPL hiện không còn hỗ trợ nhập một expression đơn lẻ và tự động in ra giá trị kết quả của nó nữa. Điều này hơi bất tiện. Hãy thêm hỗ trợ cho REPL để cho phép người dùng nhập cả statement và expression. Nếu họ nhập một statement, hãy execute nó. Nếu họ nhập một expression, hãy đánh giá và hiển thị giá trị kết quả.

2.  Có thể bạn muốn Lox rõ ràng hơn một chút về việc khởi tạo biến. Thay vì ngầm định khởi tạo biến thành `nil`, hãy biến việc truy cập một biến chưa được khởi tạo hoặc gán giá trị thành lỗi runtime, như trong ví dụ:

    ```lox
    // Không có initializer.
    var a;
    var b;

    a = "assigned";
    print a; // OK, đã được gán trước đó.

    print b; // Lỗi!
    ```

3.  Chương trình sau sẽ làm gì?

    ```lox
    var a = 1;
    {
      var a = a + 2;
      print a;
    }
    ```

    Bạn *mong đợi* nó sẽ làm gì? Nó có hoạt động như bạn nghĩ không? Code tương tự trong các ngôn ngữ khác mà bạn biết sẽ làm gì? Bạn nghĩ người dùng sẽ mong đợi nó hoạt động thế nào?

</div>

<div class="design-note">

## Ghi chú thiết kế: Khai báo biến ngầm định (Implicit Variable Declaration)

Lox có cú pháp riêng biệt cho việc khai báo một biến mới và gán giá trị cho một biến đã tồn tại. Một số ngôn ngữ gộp hai việc này lại chỉ còn cú pháp gán. Việc gán cho một biến chưa tồn tại sẽ tự động tạo ra biến đó. Điều này được gọi là **implicit variable declaration** (khai báo biến ngầm định) và tồn tại trong Python, Ruby, CoffeeScript, cùng một số ngôn ngữ khác. JavaScript có cú pháp tường minh để khai báo biến, nhưng cũng có thể tạo biến mới khi gán. Visual Basic có [tùy chọn bật hoặc tắt biến ngầm định](https://msdn.microsoft.com/en-us/library/xe53dz5w(v=vs.100).aspx).

Khi cùng một cú pháp có thể vừa gán vừa tạo biến, mỗi ngôn ngữ phải quyết định điều gì xảy ra khi không rõ người dùng muốn hành vi nào. Đặc biệt, mỗi ngôn ngữ phải chọn cách khai báo ngầm định tương tác với shadowing, và biến được khai báo ngầm định sẽ thuộc scope nào.

*   Trong Python, phép gán luôn tạo biến trong scope của hàm hiện tại, ngay cả khi có một biến cùng tên được khai báo bên ngoài hàm.

*   Ruby tránh một phần sự mơ hồ bằng cách có quy tắc đặt tên khác nhau cho biến cục bộ và biến toàn cục. Tuy nhiên, block trong Ruby (giống closure hơn là “block” kiểu C) có scope riêng, nên vẫn gặp vấn đề. Phép gán trong Ruby sẽ gán cho biến đã tồn tại bên ngoài block hiện tại nếu có biến cùng tên. Nếu không, nó tạo biến mới trong scope của block hiện tại.

*   CoffeeScript, vốn chịu ảnh hưởng nhiều từ Ruby, cũng tương tự. Nó tường minh không cho phép shadowing bằng cách quy định rằng phép gán luôn gán cho biến ở scope ngoài nếu có, kể cả lên tới scope toàn cục ngoài cùng. Nếu không, nó tạo biến trong scope của hàm hiện tại.

*   Trong JavaScript, phép gán sẽ sửa đổi biến đã tồn tại ở bất kỳ scope bao ngoài nào nếu tìm thấy. Nếu không, nó sẽ ngầm định tạo biến mới trong scope *toàn cục*.

Ưu điểm chính của khai báo ngầm định là sự đơn giản. Ít cú pháp hơn và không cần học khái niệm “khai báo”. Người dùng chỉ cần bắt đầu gán và ngôn ngữ sẽ tự xử lý.

Các ngôn ngữ kiểu tĩnh cũ như C hưởng lợi từ khai báo tường minh vì nó cho phép người dùng nói cho compiler biết kiểu của mỗi biến và lượng bộ nhớ cần cấp phát. Trong một ngôn ngữ kiểu động, có garbage collector, điều này không thực sự cần thiết, nên bạn có thể bỏ qua khai báo tường minh. Cảm giác sẽ “giống script” hơn, kiểu “bạn hiểu ý tôi mà”.

Nhưng liệu đó có phải là ý tưởng hay? Khai báo ngầm định có một số vấn đề.

*   Người dùng có thể định gán cho một biến đã tồn tại, nhưng lại gõ sai tên. Interpreter không biết điều đó, nên cứ thế tạo ra một biến mới và biến mà người dùng muốn gán vẫn giữ giá trị cũ. Điều này đặc biệt tệ trong JavaScript, nơi một lỗi gõ sẽ tạo ra biến *toàn cục*, có thể gây ảnh hưởng tới code khác.

*   JS, Ruby và CoffeeScript dùng sự tồn tại của một biến cùng tên — kể cả trong scope ngoài — để quyết định xem phép gán sẽ tạo biến mới hay gán cho biến đã tồn tại. Điều đó có nghĩa là việc thêm một biến mới ở scope bao ngoài có thể thay đổi ý nghĩa của code hiện tại. Một biến từng là cục bộ có thể âm thầm biến thành phép gán cho biến ngoài mới.

*   Trong Python, bạn có thể *muốn* gán cho một biến bên ngoài hàm hiện tại thay vì tạo biến mới trong hàm, nhưng bạn không thể.

Theo thời gian, các ngôn ngữ mà tôi biết có khai báo biến ngầm định đã thêm nhiều tính năng và độ phức tạp để xử lý những vấn đề này.

*   Khai báo ngầm định biến toàn cục trong JavaScript ngày nay được coi là một sai lầm. “Strict mode” vô hiệu hóa nó và biến nó thành lỗi compile.

*   Python thêm câu lệnh `global` để cho phép bạn tường minh gán cho biến toàn cục từ trong hàm. Sau này, khi lập trình hàm và hàm lồng nhau trở nên phổ biến hơn, họ thêm câu lệnh `nonlocal` tương tự để gán cho biến trong các hàm bao ngoài.

*   Ruby mở rộng cú pháp block để cho phép khai báo một số biến là cục bộ tường minh trong block ngay cả khi cùng tên tồn tại ở scope ngoài.

Với những điều đó, tôi nghĩ lập luận về sự đơn giản gần như không còn. Có ý kiến cho rằng khai báo ngầm định là *mặc định* đúng, nhưng cá nhân tôi thấy điều đó không thuyết phục.

Ý kiến của tôi là khai báo ngầm định hợp lý trong quá khứ khi hầu hết các ngôn ngữ script đều mang tính imperative mạnh và code khá “phẳng”. Khi lập trình viên đã quen với việc lồng sâu, lập trình hàm và closure, nhu cầu truy cập biến ở scope ngoài trở nên phổ biến hơn. Điều đó khiến khả năng người dùng gặp phải các trường hợp mơ hồ — không rõ họ muốn phép gán tạo biến mới hay dùng lại biến bao ngoài — cao hơn.

Vì vậy, tôi thích khai báo biến tường minh, và đó là lý do Lox yêu cầu điều này.

</div>