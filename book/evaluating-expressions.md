> Bạn là người tạo ra tôi, nhưng tôi mới là chủ của bạn; Hãy tuân lệnh!
>
> <cite>Mary Shelley, <em>Frankenstein</em></cite>

Nếu bạn muốn tạo đúng không khí cho chương này, hãy thử hình dung một cơn giông bão — kiểu bão xoáy cuồng loạn thích hất tung cánh cửa sổ vào đúng cao trào của câu chuyện. Có thể thêm vài tia sét cho kịch tính. Trong chương này, interpreter của chúng ta sẽ hít một hơi, mở mắt ra, và bắt đầu execute code.

<span name="spooky"></span>

<img src="image/evaluating-expressions/lightning.png" alt="Một tia sét đánh xuống một dinh thự thời Victoria. Rùng rợn!" />

<aside name="spooky">

Một dinh thự thời Victoria cũ kỹ là tùy chọn, nhưng sẽ giúp tăng thêm phần không khí.

</aside>

Có đủ mọi cách để một language implementation khiến máy tính làm theo những gì source code của người dùng yêu cầu. Nó có thể compile sang machine code, dịch sang một ngôn ngữ bậc cao khác, hoặc chuyển thành một định dạng bytecode để một virtual machine chạy. Nhưng với interpreter đầu tiên của chúng ta, ta sẽ chọn con đường đơn giản và ngắn nhất: execute trực tiếp syntax tree.

Hiện tại, parser của chúng ta chỉ hỗ trợ expression. Vậy nên, để “execute” code, ta sẽ evaluate một expression và tạo ra một giá trị. Với mỗi loại cú pháp expression mà ta có thể parse — literal, operator, v.v. — ta cần một đoạn code tương ứng biết cách evaluate cây đó và tạo ra kết quả. Điều này dẫn đến hai câu hỏi:

1. Chúng ta sẽ tạo ra những loại giá trị nào?

2. Chúng ta sẽ tổ chức các đoạn code đó ra sao?

Hãy giải quyết từng câu một…

## Representing Values

Trong Lox, <span name="value">value</span> được tạo ra bởi literal, tính toán bởi expression, và lưu trữ trong variable. Người dùng nhìn thấy chúng như các object *Lox*, nhưng chúng được implement bằng ngôn ngữ nền mà interpreter của chúng ta viết bằng nó. Điều này có nghĩa là ta phải bắc cầu giữa thế giới dynamic typing của Lox và static typing của Java. Một variable trong Lox có thể lưu trữ giá trị của bất kỳ kiểu (Lox) nào, và thậm chí có thể lưu các giá trị thuộc các kiểu khác nhau ở những thời điểm khác nhau. Vậy ta sẽ dùng kiểu Java nào để biểu diễn điều đó?

<aside name="value">

Ở đây, tôi dùng “value” và “object” gần như thay thế cho nhau.

Sau này, trong interpreter viết bằng C, chúng ta sẽ tạo một chút phân biệt giữa chúng, nhưng chủ yếu là để có các thuật ngữ riêng cho hai phần khác nhau của implementation — dữ liệu nằm trực tiếp trong biến so với dữ liệu được cấp phát trên heap. Với người dùng, hai thuật ngữ này là đồng nghĩa.

</aside>

Với một biến Java có static type như vậy, ta cũng cần có khả năng xác định kiểu giá trị mà nó đang giữ ở runtime. Khi interpreter execute toán tử `+`, nó cần biết liệu đang cộng hai số hay nối hai chuỗi. Có kiểu Java nào có thể chứa số, chuỗi, Boolean, và nhiều thứ khác không? Có kiểu nào cho ta biết kiểu runtime của nó không? Có chứ! Chính là java.lang.Object quen thuộc.

Ở những chỗ trong interpreter mà ta cần lưu một giá trị Lox, ta có thể dùng Object làm kiểu. Java có các phiên bản boxed của các kiểu nguyên thủy, tất cả đều là subclass của Object, nên ta có thể dùng chúng cho các kiểu built-in của Lox:

<table>
<thead>
<tr>
  <td>Lox type</td>
  <td>Java representation</td>
</tr>
</thead>
<tbody>
<tr>
  <td>Any Lox value</td>
  <td>Object</td>
</tr>
<tr>
  <td><code>nil</code></td>
  <td><code>null</code></td>
</tr>
<tr>
  <td>Boolean</td>
  <td>Boolean</td>
</tr>
<tr>
  <td>number</td>
  <td>Double</td>
</tr>
<tr>
  <td>string</td>
  <td>String</td>
</tr>
</tbody>
</table>


Với một giá trị có static type là Object, ta có thể xác định giá trị runtime của nó là số, chuỗi hay gì khác bằng cách dùng toán tử `instanceof` tích hợp của Java. Nói cách khác, chính representation object của <span name="jvm">JVM</span> đã tiện lợi cung cấp cho ta mọi thứ cần để implement các kiểu built-in của Lox. Sau này, khi thêm các khái niệm function, class, và instance của Lox, ta sẽ phải làm thêm một chút, nhưng Object và các lớp boxed primitive là đủ cho các kiểu mà ta cần ngay bây giờ.

<aside name="jvm">

Một việc khác ta cần làm với value là quản lý bộ nhớ của chúng, và Java cũng lo luôn phần đó. Representation object tiện lợi và một garbage collector tuyệt vời chính là lý do chính khiến ta viết interpreter đầu tiên bằng Java.

</aside>

## Evaluating Expressions

Tiếp theo, ta cần những đoạn code để implement logic evaluate cho mỗi loại expression mà ta có thể parse. Ta có thể nhét code đó vào các class của syntax tree, kiểu như một method `interpret()`. Về bản chất, ta sẽ bảo mỗi node của syntax tree: “Tự interpret chính mình đi”. Đây chính là [Interpreter design pattern](https://en.wikipedia.org/wiki/Interpreter_pattern) của nhóm Gang of Four. Đây là một pattern hay, nhưng như tôi đã nói trước đó, nó sẽ trở nên lộn xộn nếu ta nhồi đủ loại logic vào các class của tree.

Thay vào đó, ta sẽ tái sử dụng [Visitor pattern](representing-code.html#the-visitor-pattern) “ngầu” của mình. Trong chương trước, ta đã tạo class AstPrinter. Nó nhận vào một syntax tree và đệ quy duyệt qua nó, xây dựng một chuỗi và cuối cùng trả về chuỗi đó. Đó gần như chính xác là những gì một interpreter thực sự làm, chỉ khác là thay vì nối chuỗi, nó tính toán giá trị.

Bắt đầu với một class mới.

^code interpreter-class

Class này khai báo rằng nó là một visitor. Kiểu trả về của các visit method sẽ là Object — lớp gốc mà ta dùng để tham chiếu đến một giá trị Lox trong code Java. Để thỏa mãn interface Visitor, ta cần định nghĩa các visit method cho từng class của expression tree mà parser của chúng ta tạo ra. Ta sẽ bắt đầu với cái đơn giản nhất…

### Evaluating literals

Các lá của một expression tree — những mảnh cú pháp nguyên tử mà tất cả các expression khác được cấu thành từ đó — chính là <span name="leaf">literal</span>. Literal gần như đã là value rồi, nhưng sự phân biệt này rất quan trọng. Literal là *một phần cú pháp* tạo ra một value. Literal luôn xuất hiện đâu đó trong source code của người dùng. Rất nhiều value được tạo ra bởi quá trình tính toán và không hề tồn tại trực tiếp trong code. Những cái đó không phải literal. Literal thuộc về “lãnh địa” của parser. Value là một khái niệm của interpreter, thuộc về thế giới của runtime.

<aside name="leaf">

Trong [chương tiếp theo](statements-and-state.html), khi chúng ta implement variable, chúng ta sẽ thêm identifier expression, vốn cũng là các node lá.

</aside>

Vì vậy, cũng giống như khi ta chuyển một literal *token* thành một literal *syntax tree node* trong parser, bây giờ ta sẽ chuyển literal tree node đó thành một runtime value. Việc này hóa ra lại rất đơn giản.

^code visit-literal

Chúng ta đã tạo ra runtime value từ rất sớm, ngay trong lúc scanning và nhét nó vào token. Parser lấy value đó và đặt vào literal tree node, nên để evaluate một literal, ta chỉ việc lấy nó ra.

### Evaluating parentheses

Node đơn giản tiếp theo để evaluate là grouping — node mà bạn nhận được khi dùng dấu ngoặc đơn rõ ràng trong một expression.

^code visit-grouping

Một node <span name="grouping">grouping</span> có tham chiếu đến một node con bên trong, chứa expression nằm trong dấu ngoặc đơn. Để evaluate bản thân grouping expression, ta đệ quy evaluate subexpression đó và trả về kết quả.

Chúng ta dựa vào helper method này, vốn chỉ đơn giản gửi expression quay lại cho implementation visitor của interpreter:

<aside name="grouping">

Một số parser không định nghĩa node riêng cho dấu ngoặc đơn. Thay vào đó, khi parse một expression có ngoặc, chúng chỉ trả về node của expression bên trong. Chúng ta tạo node cho dấu ngoặc trong Lox vì sau này sẽ cần nó để xử lý đúng vế trái của assignment expression.

</aside>

^code evaluate

### Evaluating unary expressions

Giống như grouping, unary expression có một subexpression duy nhất mà ta phải evaluate trước. Khác biệt là bản thân unary expression sẽ làm thêm một chút việc sau đó.

^code visit-unary

Đầu tiên, ta evaluate operand expression. Sau đó, ta áp dụng unary operator lên kết quả đó. Có hai loại unary expression khác nhau, được xác định bởi loại của operator token.

Ví dụ ở đây là `-`, dùng để đảo dấu kết quả của subexpression. Subexpression này phải là một số. Vì trong Java ta không thể *statically* biết điều đó, nên ta <span name="cast">cast</span> nó trước khi thực hiện phép toán. Việc cast này diễn ra ở runtime khi `-` được evaluate. Đây chính là cốt lõi của việc một ngôn ngữ được gọi là dynamically typed.

<aside name="cast">

Có lẽ bạn đang tự hỏi chuyện gì xảy ra nếu cast thất bại. Đừng lo, chúng ta sẽ bàn đến ngay thôi.

</aside>

Bạn bắt đầu thấy cách evaluation duyệt cây một cách đệ quy. Ta không thể evaluate bản thân unary operator cho đến khi evaluate xong operand subexpression. Điều đó có nghĩa interpreter của chúng ta đang thực hiện **duyệt hậu tự (post-order traversal)** — mỗi node evaluate các node con trước khi làm công việc của chính nó.

Unary operator còn lại là logical not.

^code unary-bang (1 before, 1 after)

Phần implement rất đơn giản, nhưng “truthy” ở đây nghĩa là gì? Ta cần rẽ ngang một chút sang một trong những câu hỏi lớn của triết học phương Tây: *Thế nào là sự thật?*

### Truthiness and falsiness

OK, có thể chúng ta sẽ không thật sự bàn về câu hỏi muôn thuở này, nhưng ít nhất trong thế giới của Lox, ta cần quyết định điều gì xảy ra khi bạn dùng một thứ không phải `true` hoặc `false` trong một phép logic như `!` hoặc bất kỳ chỗ nào cần Boolean.

Chúng ta *có thể* nói đó là lỗi vì không chấp nhận chuyển đổi ngầm, nhưng hầu hết các ngôn ngữ dynamically typed không khắc khổ đến vậy. Thay vào đó, chúng lấy toàn bộ tập hợp value của mọi kiểu và chia thành hai nhóm: một nhóm được định nghĩa là “true”, “truthful” hoặc (tôi thích nhất) “truthy”, và phần còn lại là “false” hoặc “falsey”. Cách chia này phần nào mang tính tùy ý và trở nên <span name="weird">kỳ quặc</span> trong một vài ngôn ngữ.

<aside name="weird" class="bottom">

Trong JavaScript, string là truthy, nhưng string rỗng thì không. Array là truthy nhưng array rỗng thì… vẫn truthy. Số `0` là falsey, nhưng *string* `"0"` lại là truthy.

Trong Python, string rỗng là falsey giống JS, nhưng các sequence rỗng khác cũng falsey.

Trong PHP, cả số `0` và string `"0"` đều là falsey. Hầu hết các string không rỗng khác là truthy.

Bạn theo kịp chứ?

</aside>

Lox theo quy tắc đơn giản của Ruby: `false` và `nil` là falsey, còn mọi thứ khác đều là truthy. Chúng ta implement điều đó như sau:

^code is-truthy

### Evaluating binary operators

Tiếp theo là expression tree class cuối cùng: binary operator. Có một vài toán tử loại này, và chúng ta sẽ bắt đầu với các toán tử số học.

^code visit-binary

<aside name="left">

Bạn có để ý rằng ở đây chúng ta đã “đóng đinh” một chi tiết tinh tế trong semantics của ngôn ngữ không? Trong một binary expression, chúng ta evaluate các toán hạng theo thứ tự từ trái sang phải. Nếu các toán hạng này có side effect, lựa chọn này sẽ hiển thị rõ với người dùng, nên đây không chỉ đơn thuần là chi tiết implement.

Nếu muốn hai interpreter của chúng ta nhất quán (gợi ý: chúng ta muốn thế), ta cần đảm bảo clox cũng làm như vậy.

</aside>

Tôi nghĩ bạn có thể đoán được chuyện gì đang diễn ra ở đây. Khác biệt chính so với toán tử unary negation là chúng ta có hai toán hạng để evaluate.

Tôi đã bỏ qua một toán tử số học vì nó hơi đặc biệt.

^code binary-plus (3 before, 1 after)

Toán tử `+` cũng có thể được dùng để nối hai chuỗi. Để xử lý điều đó, ta không chỉ đơn giản giả định toán hạng thuộc một kiểu nhất định rồi *cast* chúng, mà ta *kiểm tra* kiểu một cách động và chọn phép toán phù hợp. Đây là lý do tại sao representation object của chúng ta cần hỗ trợ `instanceof`.

<aside name="plus">

Chúng ta hoàn toàn có thể định nghĩa một toán tử riêng cho việc nối chuỗi. Đó là cách mà Perl (`.`), Lua (`..`), Smalltalk (`,`), Haskell (`++`) và nhiều ngôn ngữ khác làm.

Tôi nghĩ sẽ thân thiện hơn với người dùng nếu Lox dùng cùng cú pháp như Java, JavaScript, Python và nhiều ngôn ngữ phổ biến khác. Điều này có nghĩa là toán tử `+` được **overload** để hỗ trợ cả cộng số và nối chuỗi. Ngay cả trong các ngôn ngữ không dùng `+` cho chuỗi, họ vẫn thường overload nó để cộng cả số nguyên và số thực dấu chấm động.

</aside>

Tiếp theo là các toán tử so sánh.

^code binary-comparison (1 before, 1 after)

Chúng về cơ bản giống với toán tử số học. Khác biệt duy nhất là trong khi toán tử số học tạo ra giá trị có kiểu giống với toán hạng (số hoặc chuỗi), thì toán tử so sánh luôn trả về Boolean.

Cặp toán tử cuối cùng là equality.

^code binary-equality

Không giống toán tử so sánh vốn yêu cầu toán hạng là số, toán tử equality hỗ trợ toán hạng thuộc bất kỳ kiểu nào, thậm chí là kiểu hỗn hợp. Bạn không thể hỏi Lox xem 3 có *nhỏ hơn* `"three"` hay không, nhưng bạn có thể hỏi nó có <span name="equal">*bằng*</span> nhau không.

<aside name="equal">

Tiết lộ trước: không đâu.

</aside>

Giống như truthiness, logic equality được tách ra thành một method riêng.

^code is-equal

Đây là một trong những chỗ mà chi tiết về cách chúng ta biểu diễn object Lox trong Java trở nên quan trọng. Chúng ta cần implement đúng khái niệm equality của *Lox*, vốn có thể khác với Java.

May mắn là hai bên khá giống nhau. Lox không thực hiện chuyển đổi ngầm trong equality và Java cũng vậy. Chúng ta chỉ cần xử lý đặc biệt `nil`/`null` để tránh NullPointerException khi gọi `equals()` trên `null`. Ngoài ra thì ổn. Method <span name="nan">`equals()`</span> của Java trên Boolean, Double và String có hành vi đúng như ta muốn cho Lox.

<aside name="nan">

Bạn nghĩ đoạn này sẽ evaluate ra gì:

```lox
(0 / 0) == (0 / 0)
```

Theo [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754), tiêu chuẩn quy định hành vi của số dấu chấm động độ chính xác kép, chia 0 cho 0 sẽ cho ra giá trị đặc biệt **NaN** (“not a number”). Lạ thay, NaN lại *không* bằng chính nó.

Trong Java, toán tử `==` trên kiểu nguyên thủy double giữ nguyên hành vi đó, nhưng method `equals()` của lớp Double thì không. Lox dùng cách sau, nên không tuân theo IEEE. Những kiểu khác biệt tinh vi như thế này chiếm một phần đáng kể trong cuộc đời của các lập trình viên ngôn ngữ.

</aside>

Và thế là xong! Đây là toàn bộ code chúng ta cần để interpret đúng một Lox expression hợp lệ. Nhưng còn một expression *không hợp lệ* thì sao? Đặc biệt là khi một subexpression evaluate ra object có kiểu sai so với phép toán đang được thực hiện?


## Runtime Errors

Tôi đã khá “liều” khi nhét các phép cast bất cứ khi nào một subexpression tạo ra một Object nhưng toán tử lại yêu cầu nó phải là số hoặc chuỗi. Những phép cast đó hoàn toàn có thể thất bại. Dù code của người dùng có sai, nếu chúng ta muốn tạo ra một ngôn ngữ <span name="fail">dễ dùng</span>, thì chúng ta có trách nhiệm xử lý lỗi đó một cách “êm đẹp”.

<aside name="fail">

Chúng ta hoàn toàn có thể chọn cách không phát hiện hoặc báo lỗi kiểu dữ liệu. Đây chính là điều C làm nếu bạn cast một con trỏ sang một kiểu không khớp với dữ liệu thực sự mà nó trỏ tới. C đạt được sự linh hoạt và tốc độ nhờ cho phép điều đó, nhưng cũng nổi tiếng là nguy hiểm. Một khi bạn diễn giải sai các bit trong bộ nhớ, thì mọi thứ đều có thể xảy ra.

Ít ngôn ngữ hiện đại chấp nhận các thao tác không an toàn như vậy. Thay vào đó, hầu hết đều **memory safe** và đảm bảo — thông qua kết hợp giữa kiểm tra tĩnh và runtime — rằng một chương trình sẽ không bao giờ diễn giải sai giá trị được lưu trong một vùng nhớ.

</aside>

Đã đến lúc nói về **runtime error**. Ở các chương trước, tôi đã tốn khá nhiều giấy mực để nói về xử lý lỗi, nhưng tất cả đều là lỗi *syntax* hoặc *static*. Những lỗi đó được phát hiện và báo trước khi *bất kỳ* code nào được execute. Runtime error là những lỗi mà semantics của ngôn ngữ yêu cầu chúng ta phải phát hiện và báo trong khi chương trình đang chạy (đúng như tên gọi).

Hiện tại, nếu một toán hạng có kiểu sai so với phép toán đang thực hiện, phép cast của Java sẽ thất bại và JVM sẽ ném ra ClassCastException. Điều đó sẽ “tháo” toàn bộ stack và thoát ứng dụng, đồng thời “phun” một Java stack trace ra cho người dùng. Đây rõ ràng không phải điều chúng ta muốn. Việc Lox được implement bằng Java nên là một chi tiết ẩn với người dùng. Thay vào đó, ta muốn họ hiểu rằng một runtime error của *Lox* đã xảy ra, và nhận được thông báo lỗi liên quan đến ngôn ngữ của chúng ta và chương trình của họ.

Tuy nhiên, hành vi của Java cũng có một điểm tốt: nó dừng execute ngay khi lỗi xảy ra. Giả sử người dùng nhập một expression như:

```lox
2 * (3 / -"muffin")
```

Bạn không thể negate một <span name="muffin">muffin</span>, nên ta cần báo runtime error tại expression `-` bên trong. Điều đó đồng nghĩa ta không thể evaluate expression `/` vì nó không có toán hạng phải hợp lệ. Tương tự với `*`. Vậy nên khi một runtime error xảy ra sâu bên trong một expression, ta cần thoát ra hoàn toàn.

<aside name="muffin">

Tôi cũng không chắc, *có* negate được muffin không nhỉ?

<img src="image/evaluating-expressions/muffin.png" alt="Một chiếc muffin, bị negate." />

</aside>

Chúng ta có thể in ra runtime error rồi hủy tiến trình và thoát ứng dụng hoàn toàn. Cách này cũng có chút “drama” — kiểu như phiên bản interpreter của ngôn ngữ lập trình thực hiện một cú “mic drop”.

Nghe cũng hấp dẫn, nhưng có lẽ ta nên làm gì đó ít “tận thế” hơn. Một runtime error cần dừng việc evaluate *expression*, nhưng không nên “giết” luôn *interpreter*. Nếu người dùng đang chạy REPL và gõ nhầm một dòng code, họ vẫn nên tiếp tục phiên làm việc và nhập thêm code sau đó.

### Detecting runtime errors

Tree-walk interpreter của chúng ta evaluate các expression lồng nhau bằng cách gọi method đệ quy, và ta cần “tháo” ra khỏi tất cả các lời gọi đó. Ném một exception trong Java là một cách tốt để làm điều này. Tuy nhiên, thay vì dùng lỗi cast sẵn có của Java, ta sẽ định nghĩa một lỗi riêng cho Lox để xử lý theo cách mình muốn.

Trước khi cast, ta sẽ tự kiểm tra kiểu của object. Ví dụ, với unary `-`, ta thêm:

^code check-unary-operand (1 before, 1 after)

Code để kiểm tra toán hạng là:

^code check-operand

Khi kiểm tra thất bại, nó sẽ ném ra một trong những lỗi này:

^code runtime-error-class

Không giống exception cast của Java, <span name="class">class</span> của chúng ta sẽ lưu token xác định vị trí trong code người dùng nơi runtime error xảy ra. Giống như với static error, điều này giúp người dùng biết cần sửa code ở đâu.

<aside name="class">

Tôi thừa nhận tên “RuntimeError” hơi gây nhầm lẫn vì Java cũng có class RuntimeException. Một điều khó chịu khi xây dựng interpreter là tên bạn chọn thường va chạm với những tên đã được ngôn ngữ implement sẵn. Cứ đợi đến khi chúng ta hỗ trợ Lox class thì biết.

</aside>

Chúng ta cần kiểm tra tương tự cho các binary operator. Vì tôi đã hứa sẽ cho bạn thấy từng dòng code cần thiết để implement interpreter, nên tôi sẽ liệt kê hết.

Greater than:

^code check-greater-operand (1 before, 1 after)

Greater than or equal to:

^code check-greater-equal-operand (1 before, 1 after)

Less than:

^code check-less-operand (1 before, 1 after)

Less than or equal to:

^code check-less-equal-operand (1 before, 1 after)

Subtraction:

^code check-minus-operand (1 before, 1 after)

Division:

^code check-slash-operand (1 before, 1 after)

Multiplication:

^code check-star-operand (1 before, 1 after)

Tất cả những cái này đều dựa vào validator này, gần như giống hệt với cái dùng cho unary:

^code check-operands


<aside name="operand">

Một lựa chọn semantics tinh tế khác: Chúng ta evaluate *cả hai* toán hạng trước khi kiểm tra kiểu của *bất kỳ* cái nào. Hãy tưởng tượng ta có một hàm `say()` in ra đối số của nó rồi trả về chính giá trị đó. Dùng hàm này, ta viết:

```lox
say("left") - say("right");
```

Interpreter của chúng ta sẽ in ra "left" và "right" trước khi báo runtime error. Ta hoàn toàn có thể quy định rằng toán hạng bên trái sẽ được kiểm tra trước khi thậm chí evaluate toán hạng bên phải.

</aside>

Toán tử cuối cùng còn lại, và lại là một ngoại lệ, chính là phép cộng. Vì `+` được overload cho cả số và chuỗi, nó đã có sẵn code để kiểm tra kiểu. Tất cả những gì ta cần làm là báo lỗi nếu không rơi vào một trong hai trường hợp hợp lệ đó.

^code string-wrong-type (3 before, 1 after)

Như vậy là ta đã phát hiện được runtime error sâu bên trong bộ evaluator. Lỗi đã được ném ra. Bước tiếp theo là viết code để bắt chúng. Để làm điều đó, ta cần kết nối class Interpreter vào class Lox chính, nơi điều khiển toàn bộ.

## Hooking Up the Interpreter

Các visit method chính là “ruột” của class Interpreter, nơi công việc thực sự diễn ra. Ta cần “bọc” một lớp bên ngoài để giao tiếp với phần còn lại của chương trình. Public API của Interpreter chỉ đơn giản là một method.

^code interpret

Method này nhận vào một syntax tree của một expression và evaluate nó. Nếu thành công, `evaluate()` trả về một object chứa giá trị kết quả. `interpret()` sẽ chuyển nó thành chuỗi và hiển thị cho người dùng. Để chuyển một Lox value thành chuỗi, ta dùng:

^code stringify

Đây là một trong những đoạn code giống như `isTruthy()`, đóng vai trò “bắc cầu” giữa cách người dùng nhìn thấy Lox object và cách chúng được biểu diễn nội bộ trong Java.

Cách làm khá đơn giản. Vì Lox được thiết kế để quen thuộc với những ai đến từ Java, nên các giá trị như Boolean trông giống hệt ở cả hai ngôn ngữ. Hai trường hợp đặc biệt là `nil`, mà ta biểu diễn bằng `null` của Java, và số.

Lox dùng số double-precision ngay cả cho giá trị nguyên. Trong trường hợp đó, chúng nên được in ra mà không có dấu thập phân. Vì Java có cả kiểu số thực và số nguyên, nó muốn bạn biết mình đang dùng loại nào. Nó thể hiện điều đó bằng cách thêm `.0` vào các double có giá trị nguyên. Chúng ta không quan tâm đến điều đó, nên ta sẽ <span name="number">cắt</span> phần đó đi.

<aside name="number">

Một lần nữa, ta xử lý trường hợp đặc biệt này với số để đảm bảo jlox và clox hoạt động giống nhau. Xử lý những góc cạnh kỳ quặc của ngôn ngữ như thế này có thể khiến bạn phát điên, nhưng đó là một phần quan trọng của công việc.

Người dùng dựa vào những chi tiết này — dù là cố ý hay vô tình — và nếu các bản implement không nhất quán, chương trình của họ sẽ hỏng khi chạy trên các interpreter khác nhau.

</aside>

### Reporting runtime errors

Nếu một runtime error bị ném ra khi evaluate expression, `interpret()` sẽ bắt nó. Điều này cho phép ta báo lỗi cho người dùng và tiếp tục chạy một cách êm đẹp. Tất cả code báo lỗi hiện tại của chúng ta nằm trong class Lox, nên ta sẽ đặt method này ở đó:

^code runtime-error-method

Chúng ta dùng token gắn với RuntimeError để cho người dùng biết dòng code nào đang được execute khi lỗi xảy ra. Tốt hơn nữa là đưa cho họ toàn bộ call stack để thấy họ *đã* đi đến đoạn code đó như thế nào. Nhưng hiện tại ta chưa có function call, nên cũng chưa cần lo.

Sau khi hiển thị lỗi, `runtimeError()` sẽ gán giá trị cho field này:

^code had-runtime-error-field (1 before, 1 after)

Field này đóng một vai trò nhỏ nhưng quan trọng.

^code check-runtime-error (4 before, 1 after)

Nếu người dùng đang chạy một <span name="repl">script Lox từ file</span> và xảy ra runtime error, ta sẽ đặt exit code khi tiến trình thoát để tiến trình gọi nó biết. Không phải ai cũng quan tâm đến “nghi thức” shell, nhưng chúng ta thì có.

<aside name="repl">

Nếu người dùng đang chạy REPL, ta không cần theo dõi runtime error. Sau khi báo lỗi, ta chỉ việc lặp lại vòng nhập và cho phép họ gõ code mới để tiếp tục.

</aside>

### Running the interpreter

Giờ chúng ta đã có một interpreter, class Lox có thể bắt đầu sử dụng nó.

^code interpreter-instance (1 before, 1 after)

Ta khai báo field này là static để các lần gọi `run()` liên tiếp trong một phiên REPL sẽ tái sử dụng cùng một interpreter. Điều này hiện tại chưa tạo ra khác biệt, nhưng sau này khi interpreter lưu trữ các biến global, chúng sẽ cần được giữ nguyên trong suốt phiên REPL.

Cuối cùng, ta xóa dòng code tạm thời từ [chương trước](parsing-expressions.html) — dòng in ra syntax tree — và thay bằng đoạn này:

^code interpreter-interpret (3 before, 1 after)

Giờ chúng ta đã có một pipeline ngôn ngữ hoàn chỉnh: scanning, parsing và execution. Chúc mừng, bạn đã có cho riêng mình một chiếc máy tính số học mini.

Như bạn thấy, interpreter hiện tại vẫn còn khá “xương xẩu”. Nhưng class Interpreter và Visitor pattern mà ta thiết lập hôm nay chính là bộ khung mà các chương sau sẽ “nhồi” thêm những phần thú vị — biến, hàm, v.v. Ngay lúc này, interpreter chưa làm được nhiều, nhưng nó đã “sống” rồi!

<img src="image/evaluating-expressions/skeleton.png" alt="Một bộ xương đang vẫy tay chào." />

<div class="challenges">

## Challenges

1.  Cho phép so sánh trên các kiểu khác ngoài số có thể hữu ích. Các toán tử này có thể có cách diễn giải hợp lý cho chuỗi. Thậm chí so sánh giữa các kiểu hỗn hợp, như `3 < "pancake"`, có thể hữu ích để hỗ trợ các cấu trúc dữ liệu có thứ tự chứa nhiều kiểu khác nhau. Hoặc cũng có thể chỉ dẫn đến lỗi và sự nhầm lẫn.

    Bạn có mở rộng Lox để hỗ trợ so sánh các kiểu khác không? Nếu có, bạn cho phép những cặp kiểu nào và định nghĩa thứ tự của chúng ra sao? Hãy giải thích lựa chọn của bạn và so sánh với các ngôn ngữ khác.

2.  Nhiều ngôn ngữ định nghĩa `+` sao cho nếu *một trong hai* toán hạng là chuỗi, toán hạng còn lại sẽ được chuyển thành chuỗi và kết quả sẽ được nối lại. Ví dụ, `"scone" + 4` sẽ cho ra `scone4`. Hãy mở rộng code trong `visitBinaryExpr()` để hỗ trợ điều đó.

3.  Hiện tại, điều gì xảy ra nếu bạn chia một số cho 0? Bạn nghĩ điều gì *nên* xảy ra? Hãy giải thích lựa chọn của bạn. Các ngôn ngữ khác mà bạn biết xử lý chia cho 0 như thế nào, và tại sao họ lại chọn cách đó?

    Hãy thay đổi implement trong `visitBinaryExpr()` để phát hiện và báo runtime error cho trường hợp này.

</div>

<div class="design-note">

## Design Note: Static and Dynamic Typing

Một số ngôn ngữ, như Java, là statically typed, nghĩa là lỗi kiểu được phát hiện và báo ở compile time trước khi bất kỳ code nào được chạy. Những ngôn ngữ khác, như Lox, là dynamically typed và hoãn việc kiểm tra lỗi kiểu đến runtime, ngay trước khi một phép toán được thực hiện. Chúng ta thường coi đây là hai thái cực trắng–đen, nhưng thực tế tồn tại một phổ liên tục giữa chúng.

Thực tế, ngay cả hầu hết các ngôn ngữ statically typed cũng thực hiện *một số* kiểm tra kiểu ở runtime. Hệ thống kiểu sẽ kiểm tra hầu hết các quy tắc kiểu một cách tĩnh, nhưng sẽ chèn thêm các kiểm tra runtime vào code sinh ra cho một số thao tác khác.

Ví dụ, trong Java, hệ thống kiểu *tĩnh* giả định rằng một biểu thức cast sẽ luôn thành công an toàn. Sau khi bạn cast một giá trị, bạn có thể coi nó là kiểu đích và không gặp lỗi compile. Nhưng downcast thì rõ ràng có thể thất bại. Lý do duy nhất mà bộ kiểm tra tĩnh có thể giả định cast luôn thành công mà không vi phạm tính soundness của ngôn ngữ là vì cast sẽ được kiểm tra *ở runtime* và ném exception nếu thất bại.

Một ví dụ tinh tế hơn là [covariant arrays](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Covariant_arrays_in_Java_and_C.23) trong Java và C#. Quy tắc subtyping tĩnh cho array cho phép những thao tác không sound. Xem ví dụ:

```java
Object[] stuff = new Integer[1];
stuff[0] = "not an int!";
```

Code này compile mà không báo lỗi. Dòng đầu tiên upcast mảng Integer và lưu nó vào biến kiểu mảng Object. Dòng thứ hai lưu một chuỗi vào một ô của mảng. Kiểu mảng Object cho phép điều đó về mặt tĩnh — string *là* Object — nhưng mảng Integer thực tế mà `stuff` trỏ tới ở runtime thì không bao giờ nên chứa string! Để tránh thảm họa này, khi bạn lưu một giá trị vào mảng, JVM sẽ thực hiện kiểm tra *runtime* để đảm bảo nó thuộc kiểu cho phép. Nếu không, nó sẽ ném ArrayStoreException.

Java hoàn toàn có thể tránh việc phải kiểm tra runtime này bằng cách không cho phép cast ở dòng đầu tiên. Nó có thể làm cho array trở thành *invariant*, nghĩa là mảng Integer *không* phải là mảng Object. Cách này sound về mặt tĩnh, nhưng lại cấm những mẫu code phổ biến và an toàn chỉ đọc từ mảng. Covariance là an toàn nếu bạn không bao giờ *ghi* vào mảng. Những mẫu này đặc biệt quan trọng cho tính tiện dụng của Java 1.0 trước khi nó hỗ trợ generics. James Gosling và các nhà thiết kế Java đã đánh đổi một chút tính an toàn tĩnh và hiệu năng — các kiểm tra khi lưu vào mảng tốn thời gian — để đổi lấy sự linh hoạt.

Hiếm có ngôn ngữ statically typed hiện đại nào không thực hiện sự đánh đổi này *ở đâu đó*. Ngay cả Haskell cũng cho phép bạn chạy code với các pattern match không đầy đủ. Nếu bạn đang thiết kế một ngôn ngữ statically typed, hãy nhớ rằng đôi khi bạn có thể cho người dùng thêm sự linh hoạt mà không hy sinh *quá nhiều* lợi ích của tính an toàn tĩnh bằng cách hoãn một số kiểm tra kiểu đến runtime.

Mặt khác, một lý do quan trọng khiến người dùng chọn ngôn ngữ statically typed là vì sự tự tin mà ngôn ngữ mang lại rằng một số loại lỗi *không bao giờ* xảy ra khi chương trình chạy. Nếu bạn hoãn quá nhiều kiểm tra kiểu đến runtime, bạn sẽ làm xói mòn niềm tin đó.

</div>