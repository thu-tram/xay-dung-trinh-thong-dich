> Không ai có quyền yêu hay ghét bất cứ điều gì nếu chưa hiểu tường tận bản chất của nó. Tình yêu lớn nảy sinh từ sự hiểu biết sâu sắc về đối tượng được yêu, và nếu bạn chỉ biết chút ít, bạn sẽ chỉ có thể yêu nó một chút hoặc chẳng yêu chút nào.
>
> <cite>Leonardo da Vinci</cite>

Chúng ta đã đi đến chương thứ mười một, và interpreter đang nằm trên máy bạn giờ đây gần như đã là một ngôn ngữ scripting hoàn chỉnh. Nó có thể cần thêm vài cấu trúc dữ liệu dựng sẵn như list và map, và chắc chắn cần một thư viện lõi cho file I/O, nhập liệu từ người dùng, v.v. Nhưng bản thân ngôn ngữ thì đã đủ dùng. Chúng ta đang có một ngôn ngữ thủ tục nhỏ, cùng “dòng họ” với BASIC, Tcl, Scheme (trừ macro), và những phiên bản đầu của Python và Lua.

Nếu đây là thập niên 80, có lẽ ta sẽ dừng lại ở đây. Nhưng ngày nay, nhiều ngôn ngữ phổ biến hỗ trợ “lập trình hướng đối tượng”. Thêm tính năng này vào Lox sẽ mang đến cho người dùng một bộ công cụ quen thuộc để viết các chương trình lớn hơn. Ngay cả khi cá nhân bạn không <span name="hate">thích</span> OOP, chương này và [chương tiếp theo](inheritance.html) sẽ giúp bạn hiểu cách người khác thiết kế và xây dựng hệ thống đối tượng.

<aside name="hate">

Nếu bạn *thực sự* ghét class, bạn có thể bỏ qua hai chương này. Chúng khá tách biệt với phần còn lại của cuốn sách. Cá nhân tôi thấy rằng việc tìm hiểu kỹ hơn về những thứ mình không thích là điều tốt. Từ xa, mọi thứ trông đơn giản, nhưng khi lại gần, các chi tiết hiện ra và tôi có được góc nhìn tinh tế hơn.

</aside>

## OOP & Class

Có ba hướng tiếp cận chính với lập trình hướng đối tượng: class, [prototype](http://gameprogrammingpatterns.com/prototype.html), và <span name="multimethods">[multimethod](https://en.wikipedia.org/wiki/Multiple_dispatch)</span>. Class xuất hiện đầu tiên và là phong cách phổ biến nhất. Với sự trỗi dậy của JavaScript (và ở mức độ nhỏ hơn là [Lua](https://www.lua.org/pil/13.4.1.html)), prototype được biết đến rộng rãi hơn trước đây. Tôi sẽ nói thêm về chúng [sau](#design-note). Với Lox, chúng ta sẽ chọn cách tiếp cận… cổ điển.

<aside name="multimethods">

Multimethod là hướng tiếp cận mà có lẽ bạn ít quen thuộc nhất. Tôi rất muốn nói nhiều hơn về chúng — tôi từng thiết kế [một ngôn ngữ “nghịch”](http://magpie-lang.org/) xoay quanh chúng và chúng *cực kỳ thú vị* — nhưng số trang của cuốn sách là có hạn. Nếu bạn muốn tìm hiểu thêm, hãy xem [CLOS](https://en.wikipedia.org/wiki/Common_Lisp_Object_System) (hệ thống đối tượng trong Common Lisp), [Dylan](https://opendylan.org/), [Julia](https://julialang.org/), hoặc [Raku](https://docs.raku.org/language/functions#Multi-dispatch).

</aside>

Vì bạn đã cùng tôi viết khoảng cả nghìn dòng Java, tôi sẽ giả định rằng bạn không cần một phần giới thiệu chi tiết về lập trình hướng đối tượng. Mục tiêu chính là gói dữ liệu cùng với code xử lý nó. Người dùng làm điều đó bằng cách khai báo một *class*:

<span name="circle"></span>

1. Cung cấp một *constructor* để tạo và khởi tạo *instance* mới của class  
2. Cung cấp cách lưu trữ và truy cập *field* trên instance  
3. Định nghĩa một tập *method* được chia sẻ bởi tất cả instance của class, hoạt động trên trạng thái của từng instance  

Đó là mức tối giản nhất. Hầu hết các ngôn ngữ hướng đối tượng, từ thời Simula, cũng có tính năng kế thừa để tái sử dụng hành vi giữa các class. Chúng ta sẽ thêm phần đó ở [chương sau](inheritance.html). Ngay cả khi bỏ qua nó, vẫn còn khá nhiều việc phải làm. Đây là một chương lớn và mọi thứ sẽ chỉ thực sự “ăn khớp” khi ta có đủ các mảnh ghép trên, nên hãy chuẩn bị tinh thần.

<aside name="circle">

<img src="image/classes/circle.png" alt="Mối quan hệ giữa class, method, instance, constructor, và field." />

Giống như “vòng tròn của sự sống”, chỉ là *không có* Sir Elton John.

</aside>

## Khai báo Class

Như thường lệ, ta sẽ bắt đầu với cú pháp. Một câu lệnh `class` giới thiệu một tên mới, nên nó nằm trong luật ngữ pháp `declaration`.

```ebnf
declaration    → classDecl
               | funDecl
               | varDecl
               | statement ;

classDecl      → "class" IDENTIFIER "{" function* "}" ;
```

Luật `classDecl` mới này dựa vào luật `function` mà ta đã định nghĩa [trước đó](functions.html#function-declarations). Nhắc lại cho bạn nhớ:

```ebnf
function       → IDENTIFIER "(" parameters? ")" block ;
parameters     → IDENTIFIER ( "," IDENTIFIER )* ;
```

Nói đơn giản, một khai báo class gồm từ khóa `class`, theo sau là tên class, rồi đến phần thân trong dấu ngoặc nhọn. Bên trong là danh sách các khai báo method. Khác với khai báo hàm, method không có từ khóa <span name="fun">`fun`</span> ở đầu. Mỗi method gồm tên, danh sách tham số và phần thân. Ví dụ:

<aside name="fun">

Không phải tôi đang ám chỉ rằng method không “fun” đâu nhé.

</aside>

```lox
class Breakfast {
  cook() {
    print "Eggs a-fryin'!";
  }

  serve(who) {
    print "Enjoy your breakfast, " + who + ".";
  }
}
```

Giống như hầu hết các ngôn ngữ dynamic typing, field không được liệt kê rõ ràng trong khai báo class. Instance chỉ là những “túi dữ liệu” lỏng lẻo và bạn có thể tự do thêm field vào chúng tùy ý bằng code mệnh lệnh thông thường.

Trong trình tạo AST của chúng ta, luật ngữ pháp `classDecl` có riêng một statement <span name="class-ast">node</span>.

^code class-ast (1 before, 1 after)

<aside name="class-ast">

Code được generated cho node mới này nằm ở [Phụ lục II](appendix-ii.html#class-statement).

</aside>

Node này lưu tên class và các method bên trong phần thân. Method được biểu diễn bằng class `Stmt.Function` hiện có mà ta dùng cho các node AST khai báo hàm. Điều đó cho chúng ta đầy đủ các thành phần trạng thái cần thiết cho một method: tên, danh sách tham số và phần thân.

Một class có thể xuất hiện ở bất kỳ đâu mà một khai báo có tên được phép, được kích hoạt bởi từ khóa `class` ở đầu.

^code match-class (1 before, 1 after)

Lệnh này gọi tới:

^code parse-class-declaration

Phần này “nặng” hơn hầu hết các phương thức parse khác, nhưng về cơ bản vẫn bám sát grammar. Chúng ta đã đọc từ khóa `class`, nên tiếp theo sẽ tìm tên class như mong đợi, rồi đến dấu ngoặc nhọn mở. Khi đã vào bên trong phần thân, ta tiếp tục parse các khai báo method cho đến khi gặp dấu ngoặc nhọn đóng. Mỗi khai báo method được parse bằng lời gọi tới `function()`, hàm mà ta đã định nghĩa trong [chương giới thiệu hàm](functions.html).

Như mọi vòng lặp mở trong parser, ta cũng kiểm tra xem có chạm tới cuối file không. Điều này sẽ không xảy ra trong code đúng, vì một class phải có dấu ngoặc nhọn đóng ở cuối, nhưng nó đảm bảo parser không bị kẹt trong vòng lặp vô hạn nếu người dùng mắc lỗi cú pháp và quên kết thúc phần thân class.

Chúng ta gói tên và danh sách method vào một node `Stmt.Class` và xong. Trước đây, ta sẽ nhảy thẳng sang interpreter, nhưng giờ cần đưa node này qua resolver trước.

^code resolver-visit-class

Chúng ta chưa cần xử lý việc resolve các method, nên hiện tại chỉ cần khai báo class bằng tên của nó. Việc khai báo class như một biến local không phổ biến, nhưng Lox cho phép, nên ta cần xử lý đúng.

Giờ ta interpret khai báo class.

^code interpreter-visit-class

Phần này trông giống cách ta execute khai báo hàm. Ta khai báo tên class trong environment hiện tại. Sau đó, ta biến *syntax node* của class thành một `LoxClass`, tức là biểu diễn *runtime* của class. Tiếp theo, ta quay lại và lưu object class vào biến vừa khai báo. Quy trình binding biến hai bước này cho phép tham chiếu tới class bên trong chính các method của nó.

Chúng ta sẽ tinh chỉnh nó trong suốt chương, nhưng bản nháp đầu tiên của `LoxClass` trông như sau:

^code lox-class

Về cơ bản chỉ là một lớp bọc quanh tên. Chúng ta thậm chí chưa lưu method. Không hữu ích lắm, nhưng nó có phương thức `toString()` để ta có thể viết một script đơn giản và kiểm tra rằng object class thực sự được parse và execute.

```lox
class DevonshireCream {
  serveOn() {
    return "Scones";
  }
}

print DevonshireCream; // In ra "DevonshireCream".
```

## Tạo Instance

Chúng ta đã có class, nhưng chúng chưa làm được gì. Lox không có method “static” để gọi trực tiếp trên class, nên nếu không có instance thực sự, class là vô dụng. Vì vậy, bước tiếp theo là instance.

Mặc dù một số cú pháp và ngữ nghĩa khá tiêu chuẩn giữa các ngôn ngữ OOP, cách tạo instance mới thì không. Ruby, theo Smalltalk, tạo instance bằng cách gọi một method trên chính object class — một cách tiếp cận <span name="turtles">đệ quy</span> duyên dáng. Một số ngôn ngữ như C++ và Java có từ khóa `new` dành riêng cho việc “sinh” object mới. Python thì “gọi” class như một hàm. (JavaScript, vốn kỳ quặc, thì kiểu như làm cả hai.)

<aside name="turtles">

Trong Smalltalk, ngay cả *class* cũng được tạo bằng cách gọi method trên một object hiện có, thường là superclass mong muốn. Nó giống như câu chuyện “rùa chồng rùa” bất tận. Cuối cùng, nó dừng lại ở một vài class “ma thuật” như `Object` và `Metaclass` mà runtime tự tạo ra *từ hư vô*.

</aside>

Tôi chọn cách tiếp cận tối giản cho Lox. Chúng ta đã có object class, và đã có lời gọi hàm, nên sẽ dùng biểu thức gọi (call expression) trên object class để tạo instance mới. Giống như một class là một hàm nhà máy tạo ra instance của chính nó. Cách này vừa gọn gàng, vừa tránh phải thêm cú pháp như `new`. Do đó, ta có thể bỏ qua phần front end và đi thẳng vào runtime.

Hiện tại, nếu bạn thử:

```lox
class Bagel {}
Bagel();
```

Bạn sẽ gặp runtime error. `visitCallExpr()` kiểm tra xem object được gọi có implement `LoxCallable` không và báo lỗi vì `LoxClass` chưa làm điều đó. *Chưa* thôi.

^code lox-class-callable (2 before, 1 after)

Việc implement interface này yêu cầu hai phương thức.

^code lox-class-call-arity

Phương thức thú vị là `call()`. Khi bạn “gọi” một class, nó sẽ tạo một `LoxInstance` mới cho class được gọi và trả về. Phương thức `arity()` giúp interpreter xác nhận rằng bạn truyền đúng số lượng argument cho callable. Hiện tại, ta sẽ quy định là không được truyền argument nào. Khi đến phần constructor do người dùng định nghĩa, ta sẽ quay lại đây.

Điều đó dẫn ta đến `LoxInstance`, biểu diễn runtime của một instance thuộc class Lox. Một lần nữa, bản cài đặt đầu tiên sẽ rất đơn giản.

^code lox-instance

Giống như `LoxClass`, nó khá sơ sài, nhưng chúng ta mới chỉ bắt đầu. Nếu muốn thử, bạn có thể chạy script sau:

```lox
class Bagel {}
var bagel = Bagel();
print bagel; // In ra "Bagel instance".
```

Chương trình này chưa làm được nhiều, nhưng ít nhất nó đã bắt đầu *làm được gì đó*.


## Thuộc tính (Properties) trên Instance

Chúng ta đã có instance, vậy giờ hãy làm cho chúng hữu ích. Ở đây ta đứng trước một ngã rẽ: có thể thêm hành vi trước — tức method — hoặc bắt đầu với trạng thái — tức property. Chúng ta sẽ chọn cách thứ hai, vì như sẽ thấy, hai thứ này gắn bó với nhau theo một cách thú vị, và sẽ dễ hiểu hơn nếu ta làm property hoạt động trước.

Lox xử lý trạng thái giống JavaScript và Python. Mỗi instance là một tập hợp mở các giá trị có tên. Method trong class của instance có thể truy cập và thay đổi property, nhưng code <span name="outside">bên ngoài</span> cũng có thể làm vậy. Property được truy cập bằng cú pháp dấu `.`.

<aside name="outside">

Cho phép code bên ngoài class trực tiếp thay đổi field của object đi ngược lại nguyên tắc OOP rằng class *đóng gói* trạng thái. Một số ngôn ngữ có lập trường nguyên tắc hơn. Trong Smalltalk, field được truy cập bằng các identifier đơn giản — về cơ bản là biến chỉ có phạm vi trong method của class. Ruby dùng ký tự `@` theo sau là tên để truy cập field trong object. Cú pháp này chỉ có ý nghĩa bên trong method và luôn truy cập trạng thái của object hiện tại.

Lox, dù tốt hay xấu, không quá “mộ đạo” với đức tin OOP của mình.

</aside>

```lox
someObject.someProperty
```

Một biểu thức theo sau bởi `.` và một identifier sẽ đọc property có tên đó từ object mà biểu thức trả về. Dấu chấm này có độ ưu tiên ngang với dấu ngoặc đơn trong lời gọi hàm, nên ta đưa nó vào grammar bằng cách thay thế luật `call` hiện tại bằng:

```ebnf
call           → primary ( "(" arguments? ")" | "." IDENTIFIER )* ;
```

Sau một biểu thức primary, ta cho phép một chuỗi bất kỳ kết hợp giữa lời gọi có ngoặc và truy cập property bằng dấu chấm. “Property access” nghe hơi dài dòng, nên từ đây ta sẽ gọi chúng là “get expression”.

### Get expression

<span name="get-ast">Node cây cú pháp</span> là:

^code get-ast (1 before, 1 after)

<aside name="get-ast">

Code sinh ra cho node mới này nằm ở [Phụ lục II](appendix-ii.html#get-expression).

</aside>

Theo grammar, code parse mới sẽ nằm trong method `call()` hiện có.

^code parse-property (3 before, 4 after)

Vòng lặp `while` bên ngoài tương ứng với dấu `*` trong luật grammar. Chúng ta “lướt” qua các token, xây dựng một chuỗi các lời gọi và get khi gặp dấu ngoặc hoặc dấu chấm, như thế này:

<img src="image/classes/zip.png" alt="Parsing một chuỗi biểu thức '.' và '()' thành AST." />

Các instance của node `Expr.Get` mới sẽ được đưa vào resolver.

^code resolver-visit-get

OK, không có gì nhiều ở đây. Vì property được tra cứu <span name="dispatch">một cách động</span>, chúng không được resolve. Trong quá trình resolve, ta chỉ đệ quy vào biểu thức bên trái dấu chấm. Việc truy cập property thực sự diễn ra trong interpreter.

<aside name="dispatch">

Bạn có thể thấy rõ rằng property dispatch trong Lox là động, vì chúng ta không xử lý tên property trong bước resolve tĩnh.

</aside>

^code interpreter-visit-get

Đầu tiên, ta đánh giá biểu thức có property đang được truy cập. Trong Lox, chỉ instance của class mới có property. Nếu object là kiểu khác như số, việc gọi getter trên nó là một runtime error.

Nếu object là `LoxInstance`, ta yêu cầu nó tra cứu property. Đã đến lúc cho `LoxInstance` có trạng thái thực sự. Một map là đủ.

^code lox-instance-fields (1 before, 2 after)

Mỗi key trong map là tên property và giá trị tương ứng là giá trị của property đó. Để tra cứu property trên một instance:

^code lox-instance-get-property

<aside name="hidden">

Tra cứu hash table cho mỗi lần truy cập field là đủ nhanh với nhiều bản cài đặt ngôn ngữ, nhưng không lý tưởng. Các VM hiệu năng cao cho ngôn ngữ như JavaScript dùng các tối ưu hóa tinh vi như “[hidden classes](http://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html)” để tránh overhead này.

Trớ trêu thay, nhiều tối ưu hóa được phát minh để làm ngôn ngữ dynamic chạy nhanh lại dựa trên quan sát rằng — ngay cả trong các ngôn ngữ đó — hầu hết code khá tĩnh về kiểu object mà nó làm việc và các field của chúng.

</aside>

Một trường hợp biên thú vị cần xử lý là khi instance không *có* property với tên đã cho. Ta có thể âm thầm trả về một giá trị giả như `nil`, nhưng kinh nghiệm của tôi với các ngôn ngữ như JavaScript cho thấy hành vi này thường che giấu bug hơn là mang lại điều gì hữu ích. Thay vào đó, ta sẽ biến nó thành runtime error.

Vì vậy, việc đầu tiên là kiểm tra xem instance thực sự có field với tên đó không. Chỉ khi đó ta mới trả về. Nếu không, ta báo lỗi.

Hãy để ý cách tôi chuyển từ “property” sang “field”. Có một sự khác biệt tinh tế giữa hai khái niệm này. Field là những phần trạng thái có tên được lưu trực tiếp trong instance. Property là những “thứ” có tên mà một get expression có thể trả về. Mọi field đều là property, nhưng như ta sẽ thấy <span name="foreshadowing">sau này</span>, không phải mọi property đều là field.

<aside name="foreshadowing">

Ồ, báo trước kìa. Rùng rợn chưa!

</aside>

Về lý thuyết, giờ ta có thể đọc property trên object. Nhưng vì chưa có cách nào để thực sự nhét trạng thái vào một instance, nên chẳng có field nào để truy cập. Trước khi thử đọc, ta phải hỗ trợ việc ghi đã.


### Biểu thức Set

Setter dùng cùng cú pháp với getter, chỉ khác là chúng xuất hiện ở phía bên trái của một phép gán.

```lox
someObject.someProperty = value;
```

Trong phần grammar, chúng ta mở rộng luật cho assignment để cho phép các identifier có dấu chấm ở phía bên trái.

```ebnf
assignment     → ( call "." )? IDENTIFIER "=" assignment
               | logic_or ;
```

Không giống getter, setter không thể chain. Tuy nhiên, tham chiếu tới `call` cho phép bất kỳ biểu thức có độ ưu tiên cao nào trước dấu chấm cuối cùng, bao gồm cả một số lượng *getter* bất kỳ, như trong ví dụ:

<img src="image/classes/setter.png" alt="breakfast.omelette.filling.meat = ham" />

Lưu ý rằng chỉ phần *cuối cùng*, `.meat`, mới là *setter*. Các phần `.omelette` và `.filling` đều là *get expression*.

Cũng giống như chúng ta có hai node AST riêng biệt cho việc truy cập biến và gán biến, ta cần một <span name="set-ast">node setter thứ hai</span> để bổ sung cho node getter.

^code set-ast (1 before, 1 after)

<aside name="set-ast">

Code sinh ra cho node mới này nằm ở [Phụ lục II](appendix-ii.html#set-expression).

</aside>

Nếu bạn không nhớ, cách chúng ta xử lý assignment trong parser hơi đặc biệt. Chúng ta không thể dễ dàng biết được một chuỗi token là phía bên trái của một phép gán cho đến khi gặp dấu `=`. Giờ đây, khi luật grammar của assignment có `call` ở bên trái, mà `call` có thể mở rộng thành những biểu thức lớn tùy ý, thì dấu `=` cuối cùng có thể cách rất xa điểm mà ta cần biết mình đang parse một assignment.

Thay vào đó, mẹo của chúng ta là parse phía bên trái như một biểu thức bình thường. Sau đó, khi bắt gặp dấu bằng phía sau nó, ta lấy biểu thức đã parse và biến nó thành node cây cú pháp phù hợp cho assignment.

Chúng ta thêm một nhánh nữa vào quá trình biến đổi đó để xử lý việc chuyển một biểu thức `Expr.Get` ở bên trái thành `Expr.Set` tương ứng.

^code assign-set (1 before, 1 after)

Vậy là xong phần parse cú pháp. Ta đưa node này qua resolver.

^code resolver-visit-set

Tương tự như `Expr.Get`, bản thân property được đánh giá một cách động, nên không có gì để resolve ở đây. Tất cả những gì cần làm là đệ quy vào hai biểu thức con của `Expr.Set`: object có property đang được gán, và giá trị sẽ gán cho nó.

Tiếp theo là interpreter.

^code interpreter-visit-set

Chúng ta đánh giá object có property đang được gán và kiểm tra xem nó có phải là `LoxInstance` không. Nếu không, đó là runtime error. Nếu có, ta đánh giá giá trị cần gán và lưu nó vào instance. Việc này dựa vào một method mới trong `LoxInstance`.

<aside name="order">

Đây là một trường hợp biên về ngữ nghĩa. Có ba thao tác riêng biệt:

1. Đánh giá object.

2. Gây ra runtime error nếu nó không phải là instance của một class.

3. Đánh giá giá trị.

Thứ tự thực hiện các bước này có thể ảnh hưởng đến hành vi mà người dùng thấy được, nghĩa là chúng ta cần chỉ rõ và đảm bảo các bản cài đặt thực hiện theo cùng một thứ tự.

</aside>

^code lox-instance-set-property

Không có gì “ma thuật” ở đây. Chúng ta đưa thẳng giá trị vào Java map nơi các field được lưu. Vì Lox cho phép tự do tạo field mới trên instance, nên không cần kiểm tra xem key đã tồn tại hay chưa.

## Method trong Class

Bạn có thể tạo instance của class và nhét dữ liệu vào chúng, nhưng bản thân class thì chưa thực sự *làm* gì cả. Instance hiện tại chỉ như những map, và tất cả instance đều na ná nhau. Để khiến chúng thực sự mang cảm giác là instance *của class*, chúng ta cần hành vi — tức method.

Parser của chúng ta vốn đã parse được khai báo method, nên phần đó ổn. Chúng ta cũng không cần thêm hỗ trợ parser mới cho lời gọi method. Ta đã có `.` (getter) và `()` (lời gọi hàm). Một “lời gọi method” đơn giản chỉ là kết hợp hai thứ đó lại.

<img src="image/classes/method.png" alt="Cây cú pháp cho 'object.method(argument)" />

Điều này dẫn đến một câu hỏi thú vị: chuyện gì xảy ra nếu tách hai biểu thức đó ra? Giả sử `method` trong ví dụ này là một method của class của `object` chứ không phải một field trên instance, thì đoạn code sau sẽ làm gì?

```lox
var m = object.method;
m(argument);
```

Chương trình này “tra cứu” method và lưu kết quả — bất kể nó là gì — vào một biến, rồi gọi object đó sau. Điều này có được phép không? Bạn có thể coi method như một hàm trên instance không?

Còn chiều ngược lại thì sao?

```lox
class Box {}

fun notMethod(argument) {
  print "called function with " + argument;
}

var box = Box();
box.function = notMethod;
box.function("argument");
```

Chương trình này tạo một instance rồi lưu một hàm vào một field của nó. Sau đó, nó gọi hàm đó bằng cùng cú pháp như gọi method. Điều này có hoạt động không?

Các ngôn ngữ khác nhau có câu trả lời khác nhau cho những câu hỏi này. Người ta có thể viết hẳn một bài luận về nó. Với Lox, chúng ta sẽ nói rằng câu trả lời cho cả hai là “có, hoạt động”. Chúng ta có vài lý do để biện minh cho điều đó. Với ví dụ thứ hai — gọi một hàm được lưu trong field — ta muốn hỗ trợ vì hàm là first-class và việc lưu chúng trong field là điều hoàn toàn bình thường.

Ví dụ đầu tiên thì mơ hồ hơn. Một lý do là người dùng thường kỳ vọng có thể “tách” một biểu thức con ra thành biến local mà không làm thay đổi ý nghĩa chương trình. Bạn có thể viết:

```lox
breakfast(omelette.filledWith(cheese), sausage);
```

Và biến nó thành:

```lox
var eggs = omelette.filledWith(cheese);
breakfast(eggs, sausage);
```

Và nó vẫn làm đúng như vậy. Tương tự, vì `.` và `()` trong một lời gọi method *là* hai biểu thức riêng biệt, nên có vẻ hợp lý khi bạn có thể tách phần *lookup* ra thành một biến rồi gọi nó <span name="callback">sau</span>. Chúng ta cần suy nghĩ kỹ về “thứ” mà bạn nhận được khi tra cứu một method là gì, và nó hoạt động thế nào, kể cả trong những trường hợp kỳ quặc như:

<aside name="callback">

Một động lực khác cho việc này là callback. Thường thì bạn muốn truyền một callback mà phần thân chỉ đơn giản là gọi một method trên một object nào đó. Có thể tra cứu method và truyền nó trực tiếp sẽ giúp bạn khỏi phải viết thủ công một hàm bọc quanh nó. So sánh:

```lox
fun callback(a, b, c) {
  object.method(a, b, c);
}

takeCallback(callback);
```

Với:

```lox
takeCallback(object.method);
```

</aside>

```lox
class Person {
  sayName() {
    print this.name;
  }
}

var jane = Person();
jane.name = "Jane";

var method = jane.sayName;
method(); // ?
```

Nếu bạn lấy một method từ một instance và gọi nó sau đó, nó có “nhớ” instance mà nó được lấy ra không? `this` bên trong method đó có còn trỏ tới object gốc không?

Đây là một ví dụ “hack não” hơn:

```lox
class Person {
  sayName() {
    print this.name;
  }
}

var jane = Person();
jane.name = "Jane";

var bill = Person();
bill.name = "Bill";

bill.sayName = jane.sayName;
bill.sayName(); // ?
```

Dòng cuối sẽ in “Bill” vì đó là instance mà ta *gọi* method thông qua, hay “Jane” vì đó là instance mà ta lấy method lần đầu?

Code tương đương trong Lua và JavaScript sẽ in “Bill”. Các ngôn ngữ đó thực ra không có khái niệm “method” rõ ràng. Mọi thứ đều kiểu như “hàm trong field”, nên không rõ `jane` “sở hữu” `sayName` hơn `bill` ở điểm nào.

Nhưng Lox có cú pháp class thực sự, nên ta biết rõ cái gì là method và cái gì là function. Vì vậy, giống như Python, C# và một số ngôn ngữ khác, chúng ta sẽ để method “bind” `this` với instance gốc khi method được lấy ra lần đầu. Python gọi <span name="bound">những thứ này</span> là **bound method**.

<aside name="bound">

Tôi biết, cái tên thật “sáng tạo”, đúng không?

</aside>

Trên thực tế, đó thường là điều bạn muốn. Nếu bạn lấy một tham chiếu tới một method trên một object nào đó để dùng làm callback sau này, bạn sẽ muốn “ghi nhớ” instance mà nó thuộc về, ngay cả khi callback đó tình flag được lưu trong một field của một object khác.

Rồi, vậy là bạn vừa phải nạp khá nhiều khái niệm ngữ nghĩa vào đầu. Tạm quên những trường hợp biên đi, chúng ta sẽ quay lại sau. Bây giờ, hãy làm cho lời gọi method cơ bản hoạt động trước đã. Chúng ta đã parse được các khai báo method bên trong thân class, nên bước tiếp theo là resolve chúng.

^code resolve-methods (1 before, 1 after)

<aside name="local">

Hiện tại, việc lưu kiểu hàm vào một biến local chưa có tác dụng gì, nhưng chẳng bao lâu nữa chúng ta sẽ mở rộng đoạn code này và khi đó nó sẽ hợp lý hơn.

</aside>

Chúng ta lặp qua các method trong thân class và gọi hàm `resolveFunction()` mà ta đã viết để xử lý khai báo hàm. Điểm khác biệt duy nhất là ta truyền vào một giá trị enum FunctionType mới.

^code function-type-method (1 before, 1 after)

Điều này sẽ quan trọng khi chúng ta resolve các biểu thức `this`. Còn bây giờ thì chưa cần lo. Phần thú vị nằm ở interpreter.

^code interpret-methods (1 before, 1 after)

Khi chúng ta execute một câu lệnh khai báo class, ta biến biểu diễn cú pháp của class — node AST của nó — thành biểu diễn runtime. Giờ, ta cũng cần làm điều đó cho các method chứa trong class. Mỗi khai báo method sẽ “nở” thành một object `LoxFunction`.

Chúng ta gom tất cả chúng lại và bọc vào một map, với key là tên method. Map này được lưu trong `LoxClass`.

^code lox-class-methods (1 before, 3 after)

Nếu instance lưu trữ trạng thái, thì class lưu trữ hành vi. `LoxInstance` có map các field, còn `LoxClass` có map các method. Dù method thuộc về class, chúng vẫn được truy cập thông qua các instance của class đó.

^code lox-instance-get-method (5 before, 2 after)

Khi tra cứu một property trên instance, nếu không <span name="shadow">tìm</span> thấy field trùng tên, ta sẽ tìm method có tên đó trong class của instance. Nếu tìm thấy, ta trả về method đó. Đây chính là lúc sự khác biệt giữa “field” và “property” trở nên có ý nghĩa. Khi truy cập một property, bạn có thể nhận được một field — một phần trạng thái được lưu trên instance — hoặc có thể gặp một method được định nghĩa trên class của instance.

Việc tra cứu method được thực hiện bằng hàm sau:

<aside name="shadow">

Việc tìm field trước ngụ ý rằng field sẽ “che khuất” method, một điểm ngữ nghĩa tinh tế nhưng quan trọng.

</aside>

^code lox-class-find-method

Bạn có thể đoán rằng hàm này sẽ trở nên thú vị hơn sau này. Hiện tại, chỉ cần một lần tra cứu map đơn giản trên bảng method của class là đủ để bắt đầu. Hãy thử nhé:

<span name="crunch"></span>

```lox
class Bacon {
  eat() {
    print "Crunch crunch crunch!";
  }
}

Bacon().eat(); // Prints "Crunch crunch crunch!".
```

<aside name="crunch">

Xin lỗi nếu bạn thích thịt xông khói mềm hơn là giòn. Cứ thoải mái chỉnh lại script cho hợp khẩu vị.

</aside>

## This

Chúng ta có thể định nghĩa cả hành vi lẫn trạng thái trên object, nhưng chúng vẫn chưa được gắn kết với nhau. Bên trong một method, ta chưa có cách nào để truy cập các field của object “hiện tại” — tức instance mà method được gọi trên đó — cũng như không thể gọi các method khác trên cùng object đó.

Để truy cập được instance đó, nó cần một <span name="i">tên</span>. Smalltalk, Ruby và Swift dùng “self”. Simula, C++, Java và nhiều ngôn ngữ khác dùng “this”. Python dùng “self” theo thông lệ, nhưng về mặt kỹ thuật bạn có thể gọi nó là gì cũng được.

<aside name="i">

“I” lẽ ra sẽ là một lựa chọn tuyệt vời, nhưng việc dùng “i” cho biến vòng lặp đã có từ trước thời OOP và bắt nguồn từ Fortran. Chúng ta là “nạn nhân” của những lựa chọn ngẫu nhiên của các bậc tiền bối.

</aside>

Với Lox, vì chúng ta thường bám theo phong cách giống Java, nên sẽ dùng từ khóa `"this"`. Bên trong thân một method, một biểu thức `this` sẽ được đánh giá thành instance mà method đó được gọi trên. Hoặc, nói chính xác hơn, vì method được truy cập và sau đó mới được gọi qua hai bước, nên `this` sẽ tham chiếu tới object mà method được *truy cập* từ đó.

Điều này khiến công việc của chúng ta khó hơn. Xem ví dụ:

```lox
class Egotist {
  speak() {
    print this;
  }
}

var method = Egotist().speak;
method();
```

Ở dòng áp chót, chúng ta lấy một tham chiếu tới method `speak()` từ một instance của class. Điều đó trả về một hàm, và hàm này cần phải “ghi nhớ” instance mà nó được lấy ra, để *sau này*, ở dòng cuối cùng, nó vẫn có thể tìm lại instance đó khi hàm được gọi.

Chúng ta cần lấy `this` tại thời điểm method được truy cập và gắn nó vào hàm theo cách nào đó để nó tồn tại lâu như ta cần. Hmm… một cách để lưu trữ thêm dữ liệu đi kèm với một hàm và tồn tại cùng nó, nghe rất giống với *closure*, đúng không?

Nếu chúng ta định nghĩa `this` như một biến ẩn trong một environment bao quanh hàm được trả về khi tra cứu method, thì các lần sử dụng `this` trong thân hàm sẽ có thể tìm thấy nó sau này. `LoxFunction` vốn đã có khả năng giữ lại environment bao quanh, nên chúng ta đã có sẵn cơ chế cần thiết.

Hãy đi qua một ví dụ để xem nó hoạt động thế nào:

```lox
class Cake {
  taste() {
    var adjective = "delicious";
    print "The " + this.flavor + " cake is " + adjective + "!";
  }
}

var cake = Cake();
cake.flavor = "German chocolate";
cake.taste(); // Prints "The German chocolate cake is delicious!".
```

Khi chúng ta lần đầu đánh giá định nghĩa class, ta tạo một `LoxFunction` cho `taste()`. Closure của nó là environment bao quanh class, trong trường hợp này là environment toàn cục. Vì vậy, `LoxFunction` mà ta lưu trong method map của class trông như sau:

<img src="image/classes/closure.png" alt="Closure ban đầu cho method." />

Khi chúng ta đánh giá biểu thức get `cake.taste`, ta tạo một environment mới, bind `this` tới object mà method được truy cập từ đó (ở đây là `cake`). Sau đó, ta tạo một `LoxFunction` *mới* với cùng phần code như bản gốc nhưng sử dụng environment mới này làm closure.

<img src="image/classes/bound-method.png" alt="Closure mới bind 'this'." />

Đây là `LoxFunction` được trả về khi đánh giá biểu thức get cho tên method. Khi hàm này sau đó được gọi bởi một biểu thức `()`, ta tạo một environment cho thân method như bình thường.

<img src="image/classes/call.png" alt="Gọi bound method và tạo environment mới cho thân method." />

Parent của environment thân hàm chính là environment mà ta đã tạo trước đó để bind `this` tới object hiện tại. Do đó, mọi lần sử dụng `this` bên trong thân hàm đều sẽ được resolve thành instance đó.

Việc tái sử dụng code environment để triển khai `this` cũng xử lý tốt các trường hợp thú vị khi method và function tương tác với nhau, ví dụ:

```lox
class Thing {
  getCallback() {
    fun localFunction() {
      print this;
    }

    return localFunction;
  }
}

var callback = Thing().getCallback();
callback();
```

Trong JavaScript chẳng hạn, việc trả về một callback từ bên trong method là khá phổ biến. Callback đó có thể muốn giữ lại và truy cập vào object gốc — giá trị `this` — mà method gắn liền với nó. Hỗ trợ hiện tại của chúng ta cho closure và chuỗi environment sẽ xử lý đúng tất cả những điều này.

Giờ hãy bắt tay vào code. Bước đầu tiên là thêm <span name="this-ast">cú pháp mới</span> cho `this`.

^code this-ast (1 before, 1 after)

<aside name="this-ast">

Code sinh ra cho node mới này nằm ở [Phụ lục II](appendix-ii.html#this-expression).

</aside>

Việc parse khá đơn giản vì đây chỉ là một token duy nhất mà lexer của chúng ta đã nhận diện là một từ khóa dành riêng.

^code parse-this (2 before, 2 after)

Bạn sẽ bắt đầu thấy `this` hoạt động giống như một biến khi chúng ta đến phần resolver.

^code resolver-visit-this

Chúng ta resolve nó giống hệt như bất kỳ biến local nào khác, dùng `"this"` làm tên cho “biến” đó. Tất nhiên, hiện tại điều này sẽ không hoạt động, vì `"this"` *không* được khai báo trong bất kỳ scope nào. Hãy sửa điều đó trong `visitClassStmt()`.

^code resolver-begin-this-scope (2 before, 1 after)

Trước khi bắt đầu resolve thân các method, chúng ta đẩy một scope mới và định nghĩa `"this"` trong đó như thể nó là một biến. Sau đó, khi xong việc, ta loại bỏ scope bao quanh đó.

^code resolver-end-this-scope (2 before, 1 after)

Giờ đây, bất cứ khi nào gặp một biểu thức `this` (ít nhất là bên trong method) nó sẽ được resolve thành một “biến local” được định nghĩa trong một scope ẩn ngay bên ngoài block của thân method.

Resolver có một *scope* mới cho `this`, nên interpreter cần tạo một *environment* tương ứng cho nó. Hãy nhớ rằng, chúng ta luôn phải giữ cho chuỗi scope của resolver và chuỗi environment liên kết của interpreter đồng bộ với nhau. Tại runtime, ta tạo environment sau khi tìm thấy method trên instance. Ta thay dòng code trước đây chỉ đơn giản trả về `LoxFunction` của method bằng đoạn này:

^code lox-instance-bind-method (1 before, 3 after)

Lưu ý lời gọi mới tới `bind()`. Nó trông như sau:

^code bind-instance

Không có gì phức tạp ở đây. Chúng ta tạo một environment mới lồng bên trong closure gốc của method. Kiểu như một closure-bên-trong-closure. Khi method được gọi, environment này sẽ trở thành parent của environment thân method.

Chúng ta khai báo `"this"` như một biến trong environment đó và bind nó với instance được truyền vào — instance mà method được truy cập từ đó. *Et voilà*, `LoxFunction` được trả về giờ đây mang theo “thế giới nhỏ” của riêng nó, nơi `"this"` được bind với object.

Nhiệm vụ còn lại là interpret các biểu thức `this`. Giống như resolver, nó tương tự như interpret một biểu thức biến.

^code interpreter-visit-this

Hãy thử ngay với ví dụ cake ở phần trước. Chỉ với chưa đến hai mươi dòng code, interpreter của chúng ta xử lý `this` bên trong method, kể cả trong những tình huống phức tạp khi nó tương tác với class lồng nhau, function bên trong method, handle tới method, v.v.

### Các cách dùng `this` không hợp lệ

Khoan đã. Chuyện gì xảy ra nếu bạn cố dùng `this` *bên ngoài* một method? Ví dụ:

```lox
print this;
```

Hoặc:

```lox
fun notAMethod() {
  print this;
}
```

Sẽ không có instance nào để `this` trỏ tới nếu bạn không ở trong một method. Chúng ta có thể gán cho nó một giá trị mặc định như `nil` hoặc biến nó thành một runtime error, nhưng rõ ràng người dùng đã mắc lỗi. Càng phát hiện và sửa lỗi sớm, họ sẽ càng vui.

Bước resolve là nơi lý tưởng để phát hiện lỗi này một cách tĩnh. Nó vốn đã phát hiện các câu lệnh `return` bên ngoài hàm. Chúng ta sẽ làm điều tương tự cho `this`. Theo phong cách của enum `FunctionType` hiện có, ta định nghĩa thêm một enum `ClassType` mới.

^code class-type (1 before, 1 after)

Đúng là nó có thể chỉ là một Boolean. Khi chúng ta đến phần kế thừa, nó sẽ có thêm giá trị thứ ba, nên giờ dùng enum là hợp lý. Ta cũng thêm một trường tương ứng, `currentClass`. Giá trị của nó cho biết ta hiện đang ở bên trong một khai báo class khi duyệt cây cú pháp hay không. Ban đầu nó là `NONE`, nghĩa là ta không ở trong class nào.

Khi bắt đầu resolve một khai báo class, ta thay đổi giá trị đó.

^code set-current-class (1 before, 1 after)

Giống như với `currentFunction`, ta lưu giá trị trước đó của trường này vào một biến local. Cách này cho phép ta tận dụng JVM để giữ một stack các giá trị `currentClass`. Nhờ vậy, ta không bị mất dấu giá trị trước đó nếu một class lồng bên trong class khác.

Khi các method đã được resolve xong, ta “pop” stack đó bằng cách khôi phục giá trị cũ.

^code restore-current-class (2 before, 1 after)

Khi resolve một biểu thức `this`, trường `currentClass` cung cấp thông tin cần thiết để báo lỗi nếu biểu thức đó không nằm bên trong thân một method.

^code this-outside-of-class (1 before, 1 after)

Điều này sẽ giúp người dùng sử dụng `this` đúng cách, và giúp chúng ta không phải xử lý việc dùng sai ở runtime trong interpreter.


## Constructor & Initializer

Giờ chúng ta đã có thể làm gần như mọi thứ với class, và khi sắp kết thúc chương này, thật kỳ lạ là chúng ta lại tập trung vào… phần khởi đầu. Method và field cho phép ta đóng gói trạng thái và hành vi lại với nhau để một object luôn *duy trì* ở trạng thái hợp lệ. Nhưng làm sao để đảm bảo một object mới tinh *bắt đầu* ở trạng thái tốt?

Để làm được điều đó, ta cần constructor. Theo tôi, đây là một trong những phần khó thiết kế nhất của một ngôn ngữ, và nếu bạn quan sát kỹ hầu hết các ngôn ngữ khác, bạn sẽ thấy những <span name="cracks">“vết nứt”</span> quanh quá trình tạo object, nơi mà các mảnh ghép thiết kế không hoàn toàn khớp nhau. Có lẽ khoảnh khắc “chào đời” vốn dĩ đã lộn xộn.

<aside name="cracks">

Một vài ví dụ: Trong Java, dù các field `final` bắt buộc phải được khởi tạo, vẫn có thể đọc chúng *trước khi* được gán giá trị. Exception — một tính năng lớn và phức tạp — được thêm vào C++ chủ yếu để cho phép báo lỗi từ constructor.

</aside>

“Xây dựng” (construct) một object thực ra gồm hai bước:

1.  Runtime <span name="allocate">*cấp phát*</span> bộ nhớ cần thiết cho một instance mới. Trong hầu hết các ngôn ngữ, thao tác này nằm ở tầng rất thấp, bên dưới những gì code của người dùng có thể truy cập.

    <aside name="allocate">

    “[placement new](https://en.wikipedia.org/wiki/Placement_syntax)” của C++ là một ví dụ hiếm hoi cho phép lập trình viên “mò” vào tận ruột gan của quá trình cấp phát.

    </aside>

2.  Sau đó, một đoạn code do người dùng cung cấp sẽ được gọi để *khởi tạo* object còn “thô” này.

Bước thứ hai này mới là điều chúng ta thường nghĩ đến khi nghe từ “constructor”, nhưng ngôn ngữ thường đã làm một số việc chuẩn bị trước khi đến đó. Thực tế, interpreter Lox của chúng ta đã lo xong phần này khi tạo một object `LoxInstance` mới.

Giờ chúng ta sẽ làm phần còn lại — khởi tạo do người dùng định nghĩa. Các ngôn ngữ có nhiều cách ký hiệu khác nhau cho đoạn code thiết lập object mới của một class. C++, Java và C# dùng một method có tên trùng với tên class. Ruby và Python gọi nó là `init()`. Cách sau ngắn gọn và dễ nhớ, nên ta sẽ dùng nó.

Trong phần cài đặt `LoxCallable` của `LoxClass`, ta thêm vài dòng code.

^code lox-class-call-initializer (2 before, 1 after)

Khi một class được gọi, sau khi `LoxInstance` được tạo, ta tìm method `"init"`. Nếu tìm thấy, ta lập tức bind và gọi nó như một lời gọi method bình thường. Danh sách đối số được truyền tiếp nguyên vẹn.

Điều này có nghĩa là ta cũng cần chỉnh lại cách class khai báo arity.

^code lox-initializer-arity (1 before, 1 after)

Nếu có initializer, arity của method đó sẽ quyết định số lượng đối số bạn phải truyền khi gọi class. Tuy nhiên, ta không *bắt buộc* class phải có initializer. Nếu không có, arity mặc định vẫn là 0.

Về cơ bản, chỉ vậy thôi. Vì ta bind method `init()` trước khi gọi, nó sẽ có quyền truy cập `this` bên trong thân hàm. Điều đó, cùng với các đối số truyền vào class, là tất cả những gì bạn cần để thiết lập instance mới theo ý muốn.

### Gọi trực tiếp init()

Như thường lệ, việc khám phá vùng ngữ nghĩa mới này lại khơi ra vài tình huống kỳ lạ. Xem ví dụ:

```lox
class Foo {
  init() {
    print this;
  }
}

var foo = Foo();
print foo.init();
```

Bạn có thể “khởi tạo lại” một object bằng cách gọi trực tiếp method `init()` của nó không? Nếu có, nó sẽ trả về gì? Một câu trả lời <span name="compromise">hợp lý</span> là `nil`, vì đó là những gì phần thân hàm trả về.

Tuy nhiên — và tôi thường không thích phải thỏa hiệp chỉ để chiều theo phần cài đặt — sẽ dễ dàng hơn nhiều cho việc triển khai constructor trong clox nếu ta quy định rằng method `init()` luôn trả về `this`, ngay cả khi được gọi trực tiếp. Để giữ cho jlox tương thích với điều này, ta thêm một chút code đặc biệt vào `LoxFunction`.

<aside name="compromise">

Có lẽ “không thích” là hơi quá. Việc để các ràng buộc và nguồn lực của phần cài đặt ảnh hưởng đến thiết kế ngôn ngữ là điều hợp lý. Một ngày chỉ có ngần ấy giờ, và nếu cắt bớt vài góc cạnh ở đây đó giúp bạn mang nhiều tính năng hơn đến tay người dùng nhanh hơn, thì đó có thể là một chiến thắng lớn cho sự hài lòng và năng suất của họ. Mấu chốt là phải xác định *những* góc nào có thể cắt mà không khiến người dùng và chính bạn trong tương lai phải nguyền rủa vì tầm nhìn ngắn hạn.

</aside>

^code return-this (2 before, 1 after)


Nếu hàm là một initializer, chúng ta sẽ ghi đè giá trị trả về thực tế và buộc nó trả về `this`. Điều này dựa vào một trường mới `isInitializer`.

^code is-initializer-field (2 before, 2 after)

Chúng ta không thể chỉ đơn giản kiểm tra xem tên của `LoxFunction` có phải là `"init"` hay không, vì người dùng có thể định nghĩa một *hàm* với tên đó. Trong trường hợp đó, sẽ *không* có `this` để trả về. Để tránh trường hợp biên kỳ quặc này, chúng ta sẽ lưu trực tiếp thông tin về việc `LoxFunction` có đại diện cho một method initializer hay không. Điều này có nghĩa là ta cần quay lại và sửa những chỗ tạo `LoxFunction`.

^code construct-function (1 before, 1 after)

Với các khai báo hàm thông thường, `isInitializer` luôn là `false`. Với method, ta sẽ kiểm tra tên.

^code interpreter-method-initializer (1 before, 1 after)

Và trong `bind()`, nơi chúng ta tạo closure bind `this` vào một method, ta truyền kèm giá trị gốc của method đó.

^code lox-function-bind-with-initializer (1 before, 1 after)

### Trả về từ init()

Chúng ta vẫn chưa xong. Từ trước đến giờ, ta giả định rằng một initializer do người dùng viết sẽ không trả về giá trị một cách tường minh, vì hầu hết constructor đều không làm vậy. Nhưng nếu người dùng thử:

```lox
class Foo {
  init() {
    return "something else";
  }
}
```

Chắc chắn nó sẽ không làm điều họ mong muốn, nên tốt nhất là biến nó thành một lỗi tĩnh. Quay lại resolver, ta thêm một trường hợp mới vào `FunctionType`.

^code function-type-initializer (1 before, 1 after)

Chúng ta dùng tên của method đang được duyệt để xác định xem mình có đang resolve một initializer hay không.

^code resolver-initializer-type (1 before, 1 after)

Khi sau đó duyệt vào một câu lệnh `return`, ta kiểm tra trường này và báo lỗi nếu trả về một giá trị từ bên trong method `init()`.

^code return-in-initializer (1 before, 1 after)

Nhưng *vẫn* chưa xong. Chúng ta đã chặn việc trả về một *giá trị* từ initializer, nhưng bạn vẫn có thể dùng một `return` sớm rỗng:

```lox
class Foo {
  init() {
    return;
  }
}
```

Thực tế, đôi khi điều này cũng hữu ích, nên ta không muốn cấm hoàn toàn. Thay vào đó, nó nên trả về `this` thay vì `nil`. Đây là một chỉnh sửa đơn giản trong `LoxFunction`.

^code early-return-this (1 before, 1 after)

Nếu chúng ta đang ở trong một initializer và execute câu lệnh `return`, thay vì trả về giá trị (luôn là `nil`), ta sẽ trả về `this`.

Phù! Đó là cả một danh sách việc phải làm, nhưng phần thưởng là interpreter nhỏ bé của chúng ta đã có thêm cả một mô hình lập trình hoàn chỉnh: class, method, field, `this`, và constructor. Ngôn ngữ “bé con” của chúng ta giờ trông đã rất trưởng thành.

<div class="challenges">

## Thử thách

1.  Chúng ta đã có method trên instance, nhưng chưa có cách định nghĩa method “static” có thể được gọi trực tiếp trên object class. Hãy thêm hỗ trợ cho chúng. Dùng từ khóa `class` đứng trước method để chỉ ra đây là static method gắn với object class.

    ```lox
    class Math {
      class square(n) {
        return n * n;
      }
    }

    print Math.square(3); // Prints "9".
    ```

    Bạn có thể giải quyết theo cách mình muốn, nhưng “[metaclass](https://en.wikipedia.org/wiki/Metaclass)” được Smalltalk và Ruby sử dụng là một cách tiếp cận đặc biệt tinh tế. *Gợi ý: Cho `LoxClass` kế thừa `LoxInstance` và bắt đầu từ đó.*

2.  Hầu hết các ngôn ngữ hiện đại hỗ trợ “getter” và “setter” — các thành viên trong class trông giống như đọc/ghi field nhưng thực tế lại execute code do người dùng định nghĩa. Hãy mở rộng Lox để hỗ trợ getter method. Chúng được khai báo không có danh sách tham số. Phần thân getter sẽ được execute khi một property có tên đó được truy cập.

    ```lox
    class Circle {
      init(radius) {
        this.radius = radius;
      }

      area {
        return 3.141592653 * this.radius * this.radius;
      }
    }

    var circle = Circle(4);
    print circle.area; // Prints roughly "50.2655".
    ```

3.  Python và JavaScript cho phép bạn tự do truy cập field của object từ bên ngoài method của nó. Ruby và Smalltalk thì đóng gói trạng thái instance: chỉ method trong class mới có thể truy cập field thô, và class sẽ quyết định trạng thái nào được lộ ra. Hầu hết các ngôn ngữ static typing cung cấp các modifier như `private` và `public` để kiểm soát phần nào của class có thể truy cập từ bên ngoài, theo từng thành viên.

    Những đánh đổi giữa các cách tiếp cận này là gì và tại sao một ngôn ngữ có thể ưu tiên cách này hơn cách kia?

</div>


<div class="design-note">

## Ghi chú thiết kế: Prototype & Sức mạnh

Trong chương này, chúng ta đã giới thiệu hai thực thể runtime mới, `LoxClass` và `LoxInstance`. Cái đầu tiên là nơi chứa hành vi của object, còn cái thứ hai là nơi lưu trữ trạng thái. Vậy nếu bạn có thể định nghĩa method ngay trên một object đơn lẻ, bên trong `LoxInstance` thì sao? Khi đó, chúng ta sẽ chẳng cần `LoxClass` nữa. `LoxInstance` sẽ là một gói hoàn chỉnh để định nghĩa cả hành vi lẫn trạng thái của một object.

Chúng ta vẫn sẽ muốn có cách — không cần class — để tái sử dụng hành vi giữa nhiều instance. Ta có thể cho phép một `LoxInstance` [*delegate*](https://en.wikipedia.org/wiki/Prototype-based_programming#Delegation) trực tiếp tới một `LoxInstance` khác để dùng lại field và method của nó, kiểu như kế thừa.

Người dùng sẽ mô hình hóa chương trình của mình như một chòm sao các object, một số trong đó delegate cho nhau để phản ánh những điểm chung. Các object được dùng làm delegate sẽ đại diện cho những object “chuẩn” hoặc “mẫu” mà các object khác tinh chỉnh lại. Kết quả là một runtime đơn giản hơn, chỉ với một cấu trúc nội bộ duy nhất: `LoxInstance`.

Đó chính là nguồn gốc của tên gọi **[prototype](https://en.wikipedia.org/wiki/Prototype-based_programming)** cho mô hình này. Nó được David Ungar và Randall Smith phát minh trong một ngôn ngữ tên là [Self](http://www.selflanguage.org/). Họ bắt đầu từ Smalltalk và thực hiện bài tập tư duy ở trên để xem có thể lược giản nó đến mức nào.

Prototype đã từng là một khái niệm học thuật trong thời gian dài — hấp dẫn, tạo ra nhiều nghiên cứu thú vị nhưng không tạo được ảnh hưởng lớn trong thế giới lập trình — cho đến khi Brendan Eich nhét prototype vào JavaScript, và rồi nó nhanh chóng thống trị thế giới. Rất nhiều (rất nhiều) <span name="words">bài viết</span> đã được viết về prototype trong JavaScript. Liệu điều đó chứng tỏ prototype là một ý tưởng xuất sắc hay gây rối — hay cả hai — vẫn là một câu hỏi bỏ ngỏ.

<aside name="words">

Bao gồm [không ít bài](http://gameprogrammingpatterns.com/prototype.html) của chính tôi.

</aside>

Tôi sẽ không bàn về việc prototype có phải là một ý tưởng hay cho một ngôn ngữ hay không. Tôi đã từng tạo ra những ngôn ngữ [dựa trên prototype](http://finch.stuffwithstuff.com/) và [dựa trên class](http://wren.io/), và quan điểm của tôi về cả hai đều phức tạp. Điều tôi muốn nói ở đây là vai trò của *sự đơn giản* trong một ngôn ngữ.

Prototype đơn giản hơn class — ít code hơn cho người triển khai ngôn ngữ phải viết, và ít khái niệm hơn cho người dùng phải học và hiểu. Điều đó có khiến chúng tốt hơn không? Những người mê ngôn ngữ như chúng ta thường có xu hướng “tôn thờ” chủ nghĩa tối giản. Cá nhân tôi nghĩ sự đơn giản chỉ là một phần của phương trình. Điều chúng ta thực sự muốn mang lại cho người dùng là *sức mạnh*, mà tôi định nghĩa như sau:

```text
power = breadth × ease ÷ complexity
```

Không yếu tố nào trong số này là thước đo số học chính xác. Tôi dùng toán học ở đây như một phép ẩn dụ, không phải để định lượng thực sự.

*   **Breadth** (độ bao quát) là phạm vi các thứ khác nhau mà ngôn ngữ cho phép bạn biểu đạt. C có độ bao quát rất lớn — nó được dùng cho mọi thứ từ hệ điều hành, ứng dụng người dùng, đến trò chơi. Các ngôn ngữ chuyên biệt như AppleScript và Matlab có độ bao quát nhỏ hơn.

*   **Ease** (độ dễ dàng) là mức độ ít nỗ lực cần bỏ ra để khiến ngôn ngữ làm điều bạn muốn. “Usability” (tính khả dụng) có thể là một thuật ngữ khác, dù nó mang nhiều hàm ý hơn tôi muốn. Các ngôn ngữ “bậc cao” thường có độ dễ dàng cao hơn các ngôn ngữ “bậc thấp”. Hầu hết ngôn ngữ đều có một “hướng” tự nhiên, nơi một số việc cảm thấy dễ diễn đạt hơn những việc khác.

*   **Complexity** (độ phức tạp) là kích thước của ngôn ngữ (bao gồm runtime, thư viện lõi, công cụ, hệ sinh thái, v.v.). Người ta thường nói về số trang trong đặc tả ngôn ngữ, hoặc số lượng từ khóa nó có. Nó là lượng kiến thức mà người dùng phải nạp vào “bộ não ướt” của mình trước khi có thể làm việc hiệu quả. Đây là phản nghĩa của sự đơn giản.

Giảm độ phức tạp *có* làm tăng sức mạnh. Mẫu số càng nhỏ, giá trị kết quả càng lớn, nên trực giác của chúng ta rằng sự đơn giản là tốt là hoàn toàn đúng. Tuy nhiên, khi giảm độ phức tạp, ta phải cẩn thận không hy sinh độ bao quát hoặc độ dễ dàng, nếu không tổng sức mạnh có thể giảm. Java sẽ là một ngôn ngữ *đơn giản* hơn nếu bỏ đi string, nhưng khi đó nó sẽ không xử lý tốt các tác vụ xử lý văn bản, và cũng không dễ để hoàn thành công việc.

Nghệ thuật ở đây là tìm ra những phần phức tạp *ngẫu nhiên* có thể bỏ đi — những tính năng và tương tác của ngôn ngữ không xứng đáng với chi phí mà chúng mang lại về độ bao quát hoặc độ dễ dàng.

Nếu người dùng muốn biểu đạt chương trình của họ theo các nhóm đối tượng, thì việc tích hợp class vào ngôn ngữ sẽ tăng độ dễ dàng khi làm điều đó, hy vọng là đủ nhiều để bù cho độ phức tạp tăng thêm. Nhưng nếu đó không phải là cách người dùng sử dụng ngôn ngữ của bạn, thì cứ mạnh dạn bỏ class đi.

</div>