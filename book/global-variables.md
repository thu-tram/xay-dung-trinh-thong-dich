
> Giá như có một phát minh nào đó có thể đóng chai ký ức, như hương thơm. Và nó sẽ không bao giờ phai, không bao giờ cũ. Rồi khi ta muốn, chỉ cần mở nút chai, và sẽ như được sống lại khoảnh khắc ấy một lần nữa.
>
> <cite>Daphne du Maurier, <em>Rebecca</em></cite>

[Chương trước](hash-tables.html) là một cuộc khám phá dài về một cấu trúc dữ liệu nền tảng, sâu và lớn trong khoa học máy tính. Nặng về lý thuyết và khái niệm. Có thể đã có đôi chút bàn luận về ký hiệu big-O và thuật toán. Chương này thì ít “trí tuệ” hơn. Không có ý tưởng lớn nào để học. Thay vào đó, chỉ là một vài tác vụ kỹ thuật thẳng thắn. Khi hoàn thành, máy ảo của chúng ta sẽ hỗ trợ biến.

Thực ra, nó sẽ chỉ hỗ trợ *biến toàn cục*. Biến cục bộ sẽ xuất hiện ở [chương sau](local-variables.html). Trong jlox, chúng ta gộp cả hai vào một chương vì dùng cùng một kỹ thuật implement cho mọi biến. Ta xây dựng một chuỗi environment, mỗi scope một environment, nối lên tận đỉnh. Đó là một cách đơn giản, gọn gàng để học cách quản lý state.

Nhưng nó cũng *chậm*. Cấp phát một hash table mới mỗi khi vào một block hoặc gọi một hàm không phải là con đường dẫn đến một VM nhanh. Xét đến việc rất nhiều code liên quan đến việc dùng biến, nếu biến chậm thì mọi thứ đều chậm. Với clox, ta sẽ cải thiện bằng cách dùng chiến lược hiệu quả hơn nhiều cho biến <span name="different">cục bộ</span>, nhưng biến toàn cục thì không dễ tối ưu như vậy.

<aside name="different">

Đây là một meta-strategy phổ biến trong các implement ngôn ngữ phức tạp. Thường thì cùng một tính năng ngôn ngữ sẽ có nhiều kỹ thuật implement khác nhau, mỗi kỹ thuật được tinh chỉnh cho các kiểu sử dụng khác nhau. Ví dụ, các VM JavaScript thường có cách biểu diễn nhanh hơn cho các object được dùng giống như instance của class so với các object có tập thuộc tính thay đổi tự do hơn. Compiler C và C++ thường có nhiều cách biên dịch câu lệnh `switch` tùy vào số lượng case và mức độ “sát nhau” của các giá trị case.

</aside>

Một chút nhắc lại về semantics của Lox: Biến toàn cục trong Lox là “late bound”, hay được resolve động. Điều này nghĩa là bạn có thể compile một đoạn code tham chiếu đến biến toàn cục trước khi nó được định nghĩa. Miễn là code đó không *execute* trước khi định nghĩa xảy ra, thì mọi thứ đều ổn. Trên thực tế, điều này nghĩa là bạn có thể tham chiếu đến các biến được định nghĩa sau trong phần thân của hàm.

```lox
fun showVariable() {
  print global;
}

var global = "after";
showVariable();
```

Code như thế này có thể trông lạ, nhưng nó hữu ích khi định nghĩa các hàm đệ quy lẫn nhau. Nó cũng thân thiện hơn với REPL. Bạn có thể viết một hàm nhỏ trong một dòng, rồi định nghĩa biến mà nó dùng ở dòng tiếp theo.

Biến cục bộ hoạt động khác. Vì khai báo biến cục bộ *luôn* xảy ra trước khi nó được dùng, VM có thể resolve chúng ngay tại compile time, ngay cả trong một compiler một-pass đơn giản. Điều đó sẽ cho phép ta dùng cách biểu diễn thông minh hơn cho biến cục bộ. Nhưng đó là chuyện của chương sau. Giờ hãy chỉ tập trung vào biến toàn cục.

## Statements

Biến được tạo ra thông qua khai báo biến, nghĩa là giờ cũng là lúc thêm hỗ trợ cho statement vào compiler của chúng ta. Nếu bạn nhớ, Lox chia statement thành hai loại. “Declaration” là những statement gán một tên mới cho một giá trị. Các loại statement khác — control flow, print, v.v. — chỉ được gọi là “statement”. Chúng ta không cho phép khai báo trực tiếp bên trong control flow statement, như thế này:

```lox
if (monday) var croissant = "yes"; // Error.
```

Cho phép điều đó sẽ dẫn đến những câu hỏi khó hiểu về phạm vi của biến. Vì vậy, giống như các ngôn ngữ khác, ta cấm điều đó về mặt cú pháp bằng cách có một grammar rule riêng cho tập con các statement *được* phép bên trong thân control flow.

```ebnf
statement      → exprStmt
               | forStmt
               | ifStmt
               | printStmt
               | returnStmt
               | whileStmt
               | block ;
```

Sau đó, ta dùng một rule riêng cho cấp cao nhất của script và bên trong block.

```ebnf
declaration    → classDecl
               | funDecl
               | varDecl
               | statement ;
```

Rule `declaration` chứa các statement khai báo tên, và cũng bao gồm `statement` để cho phép tất cả các loại statement. Vì `block` bản thân nó nằm trong `statement`, bạn có thể đặt khai báo <span name="parens">bên trong</span> một control flow construct bằng cách lồng chúng trong một block.

<aside name="parens">

Block hoạt động giống như dấu ngoặc đơn đối với expression. Một block cho phép bạn đặt các statement khai báo “độ ưu tiên thấp” vào những nơi chỉ cho phép statement không khai báo “độ ưu tiên cao” hơn.

</aside>

Trong chương này, chúng ta sẽ chỉ đề cập đến một vài statement và một declaration.

```ebnf
statement      → exprStmt
               | printStmt ;

declaration    → varDecl
               | statement ;
```

Cho đến giờ, VM của chúng ta coi một “program” là một expression duy nhất vì đó là tất cả những gì ta có thể parse và compile. Trong một implement Lox đầy đủ, một program là một chuỗi các declaration. Giờ ta đã sẵn sàng hỗ trợ điều đó.

^code compile (1 before, 1 after)

Ta tiếp tục compile các declaration cho đến khi gặp cuối file nguồn. Ta compile một declaration đơn bằng cách này:

^code declaration

Chúng ta sẽ nói về variable declaration sau trong chương, nên hiện tại, ta chỉ đơn giản chuyển tiếp sang `statement()`.

^code statement

Block có thể chứa declaration, và control flow statement có thể chứa các statement khác. Điều đó nghĩa là hai hàm này cuối cùng sẽ đệ quy. Ta có thể viết luôn phần forward declaration ngay bây giờ.

^code forward-declarations (1 before, 1 after)

### Câu lệnh Print

Trong chương này, chúng ta sẽ hỗ trợ hai loại statement. Hãy bắt đầu với câu lệnh `print`, vốn tất nhiên bắt đầu bằng token `print`. Ta phát hiện nó bằng hàm helper sau:

^code match

Bạn có thể nhận ra nó từ jlox. Nếu token hiện tại có type được chỉ định, ta consume token đó và trả về `true`. Ngược lại, ta giữ nguyên token và trả về `false`. Hàm <span name="turtles">helper</span> này được implement dựa trên một helper khác:

<aside name="turtles">

Helper chồng lên helper, tầng tầng lớp lớp!

</aside>

^code check

Hàm `check()` trả về `true` nếu token hiện tại có type được chỉ định. Có vẻ hơi <span name="read">thừa</span> khi bọc nó trong một hàm, nhưng sau này ta sẽ dùng nó nhiều, và tôi nghĩ những hàm ngắn gọn, đặt tên theo động từ như thế này giúp parser dễ đọc hơn.

<aside name="read">

Nghe thì tầm thường, nhưng parser viết tay cho các ngôn ngữ “không phải đồ chơi” có thể rất lớn. Khi bạn có hàng nghìn dòng code, một hàm tiện ích giúp rút gọn từ hai dòng xuống một và làm kết quả dễ đọc hơn hoàn toàn xứng đáng tồn tại.

</aside>

Nếu ta match được token `print`, thì ta compile phần còn lại của statement ở đây:

^code print-statement

Một câu lệnh `print` sẽ evaluate một expression và in kết quả, nên trước tiên ta parse và compile expression đó. Grammar yêu cầu có dấu chấm phẩy sau đó, nên ta consume nó. Cuối cùng, ta emit một instruction mới để in kết quả.

^code op-print (1 before, 1 after)

Khi runtime, ta execute instruction này như sau:

^code interpret-print (1 before, 1 after)

Khi interpreter gặp instruction này, nó đã execute xong code cho expression, để lại giá trị kết quả trên đỉnh stack. Giờ ta chỉ cần pop và in nó ra.

Lưu ý rằng ta không push thêm gì sau đó. Đây là một điểm khác biệt quan trọng giữa expression và statement trong VM. Mỗi instruction bytecode đều có một <span name="effect">**stack effect**</span> mô tả cách instruction đó thay đổi stack. Ví dụ, `OP_ADD` pop hai giá trị và push một, khiến stack giảm đi một phần tử so với trước.

<aside name="effect">

Stack ngắn hơn một phần tử sau `OP_ADD`, nên effect của nó là -1:

<img src="image/global-variables/stack-effect.png" alt="Stack effect của một instruction OP_ADD." />

</aside>

Bạn có thể cộng các stack effect của một chuỗi instruction để ra tổng effect của chúng. Khi cộng stack effect của chuỗi instruction được compile từ bất kỳ expression hoàn chỉnh nào, tổng sẽ là một. Mỗi expression để lại một giá trị kết quả trên stack.

Bytecode cho toàn bộ một statement có tổng stack effect bằng 0. Vì statement không tạo ra giá trị nào, nó cuối cùng sẽ để stack không đổi, dù tất nhiên nó vẫn dùng stack trong quá trình execute. Điều này quan trọng vì khi ta đến phần control flow và vòng lặp, một chương trình có thể execute một chuỗi dài statement. Nếu mỗi statement làm stack tăng hoặc giảm, cuối cùng stack có thể bị tràn hoặc hụt.

Nhân tiện khi đang ở trong vòng lặp interpreter, ta nên xóa một chút code.

^code op-return (1 before, 1 after)

Khi VM chỉ compile và evaluate một expression duy nhất, ta có một đoạn code tạm trong `OP_RETURN` để in giá trị. Giờ khi đã có statement và `print`, ta không cần nó nữa. Chúng ta đã tiến thêm một <span name="return">bước</span> đến bản implement hoàn chỉnh của clox.

<aside name="return">

Nhưng mới chỉ là một bước thôi. Chúng ta sẽ quay lại `OP_RETURN` khi thêm function. Hiện tại, nó thoát toàn bộ vòng lặp interpreter.

</aside>

Như thường lệ, một instruction mới cần được hỗ trợ trong disassembler.

^code disassemble-print (1 before, 1 after)

Đó là câu lệnh `print` của chúng ta. Nếu muốn, bạn có thể thử ngay:

```lox
print 1 + 2;
print 3 * 4;
```

Thú vị đấy chứ! OK, có thể chưa đến mức hồi hộp, nhưng giờ ta có thể viết script chứa bao nhiêu statement tùy thích, và đó là một bước tiến.


### Expression statements

Chờ đến khi bạn thấy statement tiếp theo. Nếu ta *không* thấy từ khóa `print`, thì chắc chắn ta đang nhìn vào một expression statement.

^code parse-expressions-statement (1 before, 1 after)

Nó được parse như sau:

^code expression-statement

Một “expression statement” đơn giản là một expression theo sau bởi dấu chấm phẩy. Đây là cách bạn viết một expression trong ngữ cảnh cần một statement. Thường thì điều này để bạn có thể gọi một hàm hoặc evaluate một phép gán nhằm lấy side effect của nó, như thế này:

```lox
brunch = "quiche";
eat(brunch);
```

Về mặt ngữ nghĩa, một expression statement sẽ evaluate expression và bỏ qua kết quả. Compiler encode trực tiếp hành vi đó: nó compile expression, rồi emit một instruction `OP_POP`.

^code pop-op (1 before, 1 after)

Đúng như tên gọi, instruction này pop giá trị trên đỉnh stack và bỏ nó đi.

^code interpret-pop (1 before, 1 after)

Ta cũng có thể disassemble nó.

^code disassemble-pop (1 before, 1 after)

Expression statement hiện tại chưa hữu ích lắm vì ta chưa thể tạo ra expression nào có side effect, nhưng chúng sẽ trở nên thiết yếu khi [thêm function sau này](calls-and-functions.html). <span name="majority">Phần lớn</span> statement trong code thực tế của các ngôn ngữ như C là expression statement.

<aside name="majority">

Theo thống kê của tôi, 80 trong số 149 statement ở phiên bản “compiler.c” mà chúng ta có vào cuối chương này là expression statement.

</aside>

### Error synchronization

Khi đang hoàn thiện phần việc ban đầu trong compiler, ta có thể xử lý nốt một đầu mối còn bỏ dở [từ vài chương trước](compiling-expressions.html#handling-syntax-errors). Giống như jlox, clox dùng panic mode error recovery để giảm thiểu số lượng lỗi compile dây chuyền mà nó báo. Compiler sẽ thoát khỏi panic mode khi đến một điểm đồng bộ (synchronization point). Với Lox, ta chọn ranh giới statement làm điểm đó. Giờ khi đã có statement, ta có thể implement việc đồng bộ.

^code call-synchronize (1 before, 1 after)

Nếu gặp lỗi compile khi parse statement trước đó, ta sẽ vào panic mode. Khi điều đó xảy ra, sau statement đó ta bắt đầu đồng bộ.

^code synchronize

Ta bỏ qua token một cách “vô tội vạ” cho đến khi gặp thứ trông giống ranh giới statement. Ta nhận diện ranh giới này bằng cách tìm token đứng trước có thể kết thúc một statement, như dấu chấm phẩy. Hoặc ta tìm token tiếp theo bắt đầu một statement, thường là một trong các từ khóa control flow hoặc declaration.

## Variable Declarations

Chỉ *in* thôi thì chẳng giúp ngôn ngữ của bạn giành giải gì ở <span name="fair">hội chợ</span> ngôn ngữ lập trình, nên hãy chuyển sang thứ tham vọng hơn một chút: biến. Có ba thao tác ta cần hỗ trợ:

<aside name="fair">

Tôi không thể không tưởng tượng một “hội chợ ngôn ngữ” giống như hội chợ nông thôn 4H nào đó. Những dãy gian hàng lót rơm đầy các “ngôn ngữ con non” đang *moo* và *baa* gọi nhau.

</aside>

*   Khai báo một biến mới bằng câu lệnh `var`.
*   Truy cập giá trị của một biến bằng identifier expression.
*   Lưu một giá trị mới vào biến hiện có bằng assignment expression.

Ta chưa thể làm hai thao tác cuối cho đến khi có biến, nên ta bắt đầu với khai báo.

^code match-var (1 before, 2 after)

Hàm parse placeholder mà ta phác thảo cho rule grammar declaration giờ đã có phần xử lý thực sự. Nếu match được token `var`, ta nhảy đến đây:

^code var-declaration

Từ khóa này theo sau bởi tên biến. Việc này được compile bởi `parseVariable()`, mà ta sẽ nói tới ngay sau đây. Sau đó, ta tìm dấu `=` theo sau bởi một initializer expression. Nếu người dùng không khởi tạo biến, compiler sẽ ngầm định khởi tạo nó thành <span name="nil">`nil`</span> bằng cách emit instruction `OP_NIL`. Dù theo cách nào, ta cũng mong statement kết thúc bằng dấu chấm phẩy.

<aside name="nil" class="bottom">

Về cơ bản, compiler sẽ “desugar” một khai báo biến như:

```lox
var a;
```

thành:

```lox
var a = nil;
```

Code nó tạo ra cho trường hợp đầu tiên giống hệt với trường hợp thứ hai.

</aside>

Ở đây có hai hàm mới để làm việc với biến và identifier. Đây là hàm đầu tiên:

^code parse-variable (2 before)

Hàm này yêu cầu token tiếp theo phải là một identifier, nó sẽ consume token đó và chuyển sang đây:

^code identifier-constant (2 before)

Hàm này nhận token được truyền vào và thêm lexeme của nó vào constant table của chunk dưới dạng string. Sau đó, nó trả về chỉ số của constant đó trong constant table.

Biến toàn cục được tra cứu *theo tên* ở runtime. Điều đó nghĩa là VM — vòng lặp bytecode interpreter — cần truy cập được tên này. Một string đầy đủ thì quá lớn để nhét trực tiếp vào luồng bytecode như một toán hạng. Thay vào đó, ta lưu string vào constant table và instruction sẽ tham chiếu tới tên bằng chỉ số của nó trong bảng.

Hàm này trả về chỉ số đó cho `varDeclaration()`, hàm này sau đó chuyển nó tới đây:

^code define-variable

<span name="helper">Hàm này</span> xuất ra instruction bytecode định nghĩa biến mới và lưu giá trị khởi tạo của nó. Chỉ số của tên biến trong constant table chính là toán hạng của instruction. Như thường lệ trong một stack-based VM, ta emit instruction này sau cùng. Ở runtime, ta execute code cho phần khởi tạo biến trước, để lại giá trị trên stack. Sau đó, instruction này lấy giá trị đó và lưu lại để dùng sau.

<aside name="helper">

Tôi biết một số hàm này trông có vẻ khá vô nghĩa vào lúc này. Nhưng chúng sẽ hữu ích hơn khi ta thêm nhiều tính năng ngôn ngữ để làm việc với tên. Cả function và class declaration đều khai báo biến mới, và variable/assignment expression sẽ truy cập chúng.

</aside>

Bên phía runtime, ta bắt đầu với instruction mới này:

^code define-global-op (1 before, 1 after)

Nhờ có hash table tiện dụng, phần implement không quá khó.

^code interpret-define-global (1 before, 1 after)

Ta lấy tên biến từ constant table. Sau đó, ta <span name="pop">lấy</span> giá trị từ đỉnh stack và lưu nó vào hash table với tên đó làm key.

<aside name="pop">

Lưu ý rằng ta không *pop* giá trị cho đến *sau khi* thêm nó vào hash table. Điều này đảm bảo VM vẫn có thể tìm thấy giá trị nếu garbage collection được kích hoạt ngay giữa lúc thêm vào hash table. Đây là khả năng hoàn toàn có thể xảy ra vì hash table cần cấp phát động khi resize.

</aside>

Đoạn code này không kiểm tra xem key đã tồn tại trong bảng chưa. Lox khá “thoáng” với biến toàn cục và cho phép bạn định nghĩa lại chúng mà không báo lỗi. Điều này hữu ích trong phiên REPL, nên VM hỗ trợ bằng cách đơn giản ghi đè giá trị nếu key đã tồn tại trong hash table.

Có thêm một macro helper nhỏ:

^code read-string (1 before, 1 after)

Nó đọc một toán hạng một byte từ bytecode chunk. Nó coi đó là chỉ số trong constant table của chunk và trả về string ở chỉ số đó. Nó không kiểm tra giá trị *có* phải là string hay không — chỉ đơn giản cast nó. Điều này an toàn vì compiler không bao giờ emit instruction tham chiếu tới một constant không phải string.

Vì muốn giữ “vệ sinh” về mặt từ vựng, ta cũng undefine macro này ở cuối hàm interpret.

^code undef-read-string (1 before, 1 after)

Tôi cứ nói “hash table” nhưng thực ra ta chưa có cái nào. Ta cần một nơi để lưu các biến toàn cục này. Vì muốn chúng tồn tại suốt thời gian clox chạy, ta lưu chúng ngay trong VM.

^code vm-globals (1 before, 1 after)

Giống như với string table, ta cần khởi tạo hash table về trạng thái hợp lệ khi VM khởi động.

^code init-globals (1 before, 1 after)

Và ta <span name="tear">giải phóng</span> nó khi thoát.

<aside name="tear">

Quá trình thoát sẽ giải phóng mọi thứ, nhưng thật thiếu chỉn chu nếu phải nhờ hệ điều hành dọn dẹp mớ hỗn độn của chúng ta.

</aside>

^code free-globals (1 before, 1 after)

Như thường lệ, ta cũng muốn có thể disassemble instruction mới này.

^code disassemble-define-global (1 before, 1 after)

Và với điều đó, ta đã có thể định nghĩa biến toàn cục. Tất nhiên, người dùng chưa *nhận ra* là họ đã làm vậy, vì họ chưa thể *dùng* chúng. Vậy nên, hãy sửa điều đó ở bước tiếp theo.



## Đọc giá trị biến (Reading Variables)

Giống như trong mọi ngôn ngữ lập trình, ta truy cập giá trị của một biến thông qua tên của nó. Ta kết nối các identifier token với expression parser tại đây:

^code table-identifier (1 before, 1 after)

Điều này gọi tới hàm parser mới:

^code variable-without-assign

Giống như với khai báo biến, ở đây cũng có một vài hàm helper nhỏ trông có vẻ vô nghĩa lúc này nhưng sẽ hữu ích hơn ở các chương sau. Tôi hứa đấy.

^code read-named-variable

Hàm này gọi lại `identifierConstant()` như trước để lấy identifier token được truyền vào và thêm lexeme của nó vào constant table của chunk dưới dạng string. Việc còn lại là emit một instruction để load biến toàn cục có tên đó. Đây là instruction đó:

^code get-global-op (1 before, 1 after)

Bên phía interpreter, phần implement phản chiếu `OP_DEFINE_GLOBAL`.

^code interpret-get-global (1 before, 1 after)

Ta lấy chỉ số trong constant table từ toán hạng của instruction và lấy tên biến. Sau đó, ta dùng tên này làm key để tra giá trị của biến trong globals hash table.

Nếu key không tồn tại trong hash table, nghĩa là biến toàn cục đó chưa bao giờ được định nghĩa. Đây là lỗi runtime trong Lox, nên ta báo lỗi và thoát khỏi vòng lặp interpreter nếu điều đó xảy ra. Ngược lại, ta lấy giá trị và push nó lên stack.

^code disassemble-get-global (1 before, 1 after)

Thêm một chút disassemble nữa là xong. Interpreter của chúng ta giờ có thể chạy code như thế này:

```lox
var beverage = "cafe au lait";
var breakfast = "beignets with " + beverage;
print breakfast;
```

Giờ chỉ còn một thao tác nữa.

## Gán giá trị (Assignment)

Xuyên suốt cuốn sách này, tôi đã cố gắng giữ cho bạn đi trên một con đường khá an toàn và dễ dàng. Tôi không né tránh các *vấn đề* khó, nhưng tôi cố không làm cho *giải pháp* phức tạp hơn mức cần thiết. Tiếc là, một số lựa chọn thiết kế khác trong <span name="jlox">bytecode</span> compiler của chúng ta khiến việc implement assignment trở nên phiền phức.

<aside name="jlox">

Nếu bạn còn nhớ, assignment trong jlox khá dễ.

</aside>

Bytecode VM của chúng ta dùng compiler một-pass. Nó parse và sinh bytecode ngay lập tức mà không qua bất kỳ AST trung gian nào. Ngay khi nhận ra một phần cú pháp, nó sẽ emit code cho phần đó. Assignment thì không tự nhiên phù hợp với cách này. Xem ví dụ:

```lox
menu.brunch(sunday).beverage = "mimosa";
```

Trong đoạn code này, parser không nhận ra `menu.brunch(sunday).beverage` là mục tiêu của một phép gán (assignment target) chứ không phải một expression thông thường cho đến khi gặp dấu `=`, tức là nhiều token sau `menu`. Lúc đó, compiler đã emit bytecode cho toàn bộ phần đó rồi.

Tuy nhiên, vấn đề không nghiêm trọng như có vẻ. Hãy xem parser nhìn ví dụ này thế nào:

<img src="image/global-variables/setter.png" alt="Câu lệnh 'menu.brunch(sunday).beverage = &quot;mimosa&quot;', cho thấy 'menu.brunch(sunday)' là một expression." />

Mặc dù phần `.beverage` không được compile như một get expression, mọi thứ bên trái dấu `.` vẫn là một expression, với semantics thông thường. Phần `menu.brunch(sunday)` có thể compile và execute như bình thường.

May mắn cho chúng ta, sự khác biệt về semantics ở phía bên trái của một assignment chỉ xuất hiện ở phần cuối cùng của chuỗi token, ngay trước dấu `=`. Dù receiver của một setter có thể là một expression dài tùy ý, phần có hành vi khác với get expression chỉ là identifier ở cuối, ngay trước dấu `=`. Ta không cần lookahead nhiều để nhận ra `beverage` nên được compile như một set expression thay vì getter.

Với biến thì còn dễ hơn, vì chúng chỉ là một identifier đơn lẻ trước dấu `=`. Ý tưởng là ngay *trước khi* compile một expression có thể được dùng làm assignment target, ta sẽ kiểm tra xem token tiếp theo có phải là `=` không. Nếu có, ta compile nó như một assignment hoặc setter thay vì variable access hoặc getter.

Hiện tại ta chưa có setter để lo, nên tất cả những gì cần xử lý chỉ là biến.

^code named-variable (1 before, 1 after)

Trong hàm parse cho identifier expression, ta tìm dấu bằng sau identifier. Nếu tìm thấy, thay vì emit code cho variable access, ta compile giá trị được gán và sau đó emit một assignment instruction.

Đây là instruction cuối cùng cần thêm trong chương này.

^code set-global-op (1 before, 1 after)

Như bạn mong đợi, hành vi runtime của nó tương tự như khi định nghĩa biến mới.

^code interpret-set-global (1 before, 1 after)

Điểm khác biệt chính là điều gì xảy ra khi key chưa tồn tại trong globals hash table. Nếu biến chưa được định nghĩa, việc gán giá trị cho nó là lỗi runtime. Lox [không cho phép khai báo biến ngầm định](implicit).

<aside name="delete">

Lệnh gọi `tableSet()` sẽ lưu giá trị vào bảng biến toàn cục ngay cả khi biến chưa được định nghĩa trước đó. Điều này có thể thấy trong phiên REPL, vì nó vẫn tiếp tục chạy ngay cả sau khi báo lỗi runtime. Vì vậy, ta cũng xóa giá trị “xác sống” đó khỏi bảng.

</aside>

Điểm khác biệt còn lại là việc gán giá trị cho một biến **không** pop giá trị đó khỏi stack. Hãy nhớ rằng, assignment là một expression, nên nó cần để lại giá trị đó trên stack trong trường hợp phép gán này được lồng bên trong một expression lớn hơn.

[implicit]: statements-and-state.html#design-note

Thêm một chút phần disassembly:

^code disassemble-set-global (2 before, 1 after)

Vậy là xong rồi, đúng không? Ừm… chưa hẳn. Chúng ta đã mắc một lỗi! Hãy xem thử:

```lox
a * b = c + d;
```

Theo grammar của Lox, `=` có độ ưu tiên thấp nhất, nên đoạn này sẽ được parse đại khái như:

<img src="image/global-variables/ast-good.png" alt="Cách parse mong đợi, giống '(a * b) = (c + d)'." />

Rõ ràng, `a * b` không phải là một assignment target <span name="do">hợp lệ</span>, nên đây phải là lỗi cú pháp. Nhưng parser của chúng ta lại làm như sau:

<aside name="do">

Sẽ thật “điên rồ” nếu `a * b` *lại* là một assignment target hợp lệ, đúng không? Bạn có thể tưởng tượng một ngôn ngữ kiểu đại số, cố gắng chia giá trị được gán ra theo cách hợp lý nào đó và phân phối nó cho `a` và `b`… có lẽ đó sẽ là một ý tưởng tồi tệ.

</aside>

1.  Đầu tiên, `parsePrecedence()` parse `a` bằng prefix parser `variable()`.
2.  Sau đó, nó bước vào vòng lặp parse infix.
3.  Nó gặp `*` và gọi `binary()`.
4.  Hàm này đệ quy gọi `parsePrecedence()` để parse toán hạng bên phải.
5.  Lần này, `variable()` lại được gọi để parse `b`.
6.  Bên trong lời gọi `variable()` này, nó tìm dấu `=` ở phía sau. Nó thấy một dấu và parse phần còn lại của dòng như một assignment.

Nói cách khác, parser nhìn đoạn code trên như:

<img src="image/global-variables/ast-bad.png" alt="Cách parse thực tế, giống 'a * (b = c + d)'." />

Chúng ta đã làm hỏng phần xử lý precedence vì `variable()` không xét đến độ ưu tiên của expression bao quanh biến đó. Nếu biến này tình cờ nằm ở phía bên phải của một toán tử infix, hoặc là toán hạng của một toán tử unary, thì expression bao quanh đó có precedence quá cao để cho phép dấu `=`.

Để sửa, `variable()` chỉ nên tìm và consume dấu `=` nếu nó đang ở trong ngữ cảnh của một expression có precedence thấp. Phần code biết precedence hiện tại, hợp lý thay, là `parsePrecedence()`. Hàm `variable()` không cần biết chính xác cấp độ precedence, nó chỉ cần biết precedence đủ thấp để cho phép assignment, nên ta truyền thông tin đó vào dưới dạng một giá trị Boolean.

^code prefix-rule (4 before, 2 after)

Vì assignment là expression có precedence thấp nhất, thời điểm duy nhất ta cho phép assignment là khi đang parse một assignment expression hoặc một top-level expression như trong expression statement. Cờ này được truyền tới hàm parser ở đây:

^code variable

Hàm này truyền nó qua một tham số mới:

^code named-variable-signature (1 after)

Và cuối cùng sử dụng nó ở đây:

^code named-variable-can-assign (2 before, 1 after)

Khá nhiều “đường ống” chỉ để đưa đúng **một bit** dữ liệu tới đúng chỗ trong compiler, nhưng cuối cùng nó cũng tới. Nếu biến nằm bên trong một expression có precedence cao hơn, `canAssign` sẽ là `false` và đoạn code này sẽ bỏ qua dấu `=` ngay cả khi nó có ở đó. Sau đó `namedVariable()` trả về, và luồng execute cuối cùng quay lại `parsePrecedence()`.

Rồi sao nữa? Compiler sẽ làm gì với ví dụ hỏng trước đó? Lúc này, `variable()` sẽ không consume dấu `=`, nên nó sẽ là token hiện tại. Compiler quay lại `parsePrecedence()` từ prefix parser `variable()` và sau đó cố gắng bước vào vòng lặp parse infix. Không có hàm parse nào gắn với `=`, nên nó bỏ qua vòng lặp đó.

Rồi `parsePrecedence()` lặng lẽ trả về cho caller. Điều này cũng không đúng. Nếu dấu `=` không được consume như một phần của expression, sẽ không có gì khác consume nó. Đây là một lỗi và ta nên báo lỗi.

^code invalid-assign (2 before, 1 after)

Với thay đổi này, chương trình sai trước đó sẽ nhận lỗi compile đúng như mong đợi. OK, *giờ* thì xong chưa? Vẫn chưa hẳn. Bạn thấy đấy, ta đang truyền một tham số vào một trong các hàm parse. Nhưng các hàm này được lưu trong một bảng con trỏ hàm, nên tất cả các hàm parse đều phải có cùng kiểu. Dù hầu hết các hàm parse không hỗ trợ việc được dùng làm assignment target — setter là trường hợp <span name="index">duy nhất</span> khác — nhưng C compiler “thân thiện” của chúng ta yêu cầu tất cả chúng đều phải nhận tham số này.

<aside name="index">

Nếu Lox có mảng và toán tử subscript như `array[index]` thì một infix `[` cũng sẽ cho phép assignment để hỗ trợ `array[index] = value`.

</aside>

Vậy là chúng ta sẽ kết thúc chương này với một chút “việc tay chân”. Đầu tiên, hãy truyền flag này vào các hàm parse infix.

^code infix-rule (1 before, 1 after)

Rồi sau này ta sẽ cần nó cho setter. Tiếp theo, ta sửa typedef cho kiểu hàm.

^code parse-fn-type (2 before, 2 after)

Và thêm một loạt code khá nhàm chán để chấp nhận tham số này trong tất cả các hàm parse hiện có. Ở đây:

^code binary (1 after)

Và ở đây:

^code parse-literal (1 after)

Và ở đây:

^code grouping (1 after)

Và ở đây:

^code number (1 after)

Và cả ở đây nữa:

^code string (1 after)

Và cuối cùng:

^code unary (1 after)

Phù! Giờ ta lại có một chương trình C có thể compile. Chạy nó lên và bạn có thể execute đoạn này:

```lox
var breakfast = "beignets";
var beverage = "cafe au lait";
breakfast = "beignets with " + beverage;

print breakfast;
```

Nó bắt đầu trông giống code thật của một ngôn ngữ lập trình thực thụ rồi đấy!

<div class="challenges">

## Thử thách

1.  Compiler thêm tên biến toàn cục vào constant table dưới dạng string mỗi khi gặp một identifier. Nó tạo một constant mới mỗi lần, ngay cả khi tên biến đó đã tồn tại ở một slot trước đó trong constant table. Điều này lãng phí trong trường hợp cùng một biến được tham chiếu nhiều lần bởi cùng một hàm. Điều đó cũng làm tăng khả năng lấp đầy constant table và hết slot, vì ta chỉ cho phép tối đa 256 constant trong một chunk.

    Hãy tối ưu điều này. Việc tối ưu của bạn ảnh hưởng thế nào đến hiệu năng của compiler so với runtime? Đây có phải là sự đánh đổi hợp lý không?

2.  Việc tra cứu biến toàn cục theo tên trong hash table mỗi lần sử dụng khá chậm, ngay cả với một hash table tốt. Bạn có thể nghĩ ra cách nào hiệu quả hơn để lưu trữ và truy cập biến toàn cục mà không thay đổi semantics không?

3.  Khi chạy trong REPL, người dùng có thể viết một hàm tham chiếu tới một biến toàn cục chưa biết. Sau đó, ở dòng tiếp theo, họ khai báo biến đó. Lox nên xử lý trường hợp này một cách “êm” bằng cách không báo lỗi compile “unknown variable” khi hàm vừa được định nghĩa.

    Nhưng khi người dùng chạy một *script* Lox, compiler có quyền truy cập toàn bộ nội dung chương trình trước khi bất kỳ code nào được chạy. Xem chương trình này:

    ```lox
    fun useVar() {
      print oops;
    }

    var ooops = "too many o's!";
    ```

    Ở đây, ta có thể biết một cách tĩnh rằng `oops` sẽ không được định nghĩa vì *không* có khai báo nào của biến toàn cục đó trong chương trình. Lưu ý rằng `useVar()` cũng không bao giờ được gọi, nên dù biến không được định nghĩa, cũng sẽ không có lỗi runtime vì nó cũng không bao giờ được dùng.

    Ta có thể báo những lỗi như thế này ngay tại compile time, ít nhất là khi chạy từ script. Bạn có nghĩ ta nên làm vậy không? Hãy giải thích lý do. Các ngôn ngữ scripting khác mà bạn biết xử lý thế nào?

</div>