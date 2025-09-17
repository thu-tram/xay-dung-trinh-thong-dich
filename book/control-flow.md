> Logic, giống như whiskey, sẽ mất tác dụng có lợi nếu dùng quá nhiều.
>
> <cite>Edward John Moreton Drax Plunkett, Lord Dunsany</cite>

So với [chương trước](statements-and-state.html) — một cuộc chạy marathon đầy mệt mỏi — thì hôm nay giống như một buổi dạo chơi nhẹ nhàng giữa cánh đồng hoa cúc. Nhưng dù công việc khá dễ, phần thưởng lại bất ngờ lớn.

Hiện tại, interpreter của chúng ta chẳng khác gì một chiếc máy tính bỏ túi. Một chương trình Lox chỉ có thể thực hiện một lượng công việc cố định trước khi kết thúc. Muốn nó chạy lâu gấp đôi, bạn phải viết source code dài gấp đôi. Chúng ta sắp sửa thay đổi điều đó. Trong chương này, interpreter của chúng ta sẽ tiến một bước lớn để gia nhập “giải đấu lớn” của các ngôn ngữ lập trình: *Turing-completeness*.

## Máy Turing (Tóm tắt)

Vào đầu thế kỷ trước, các nhà toán học đã vấp phải một loạt <span name="paradox">nghịch lý</span> rối rắm khiến họ bắt đầu nghi ngờ sự vững chắc của nền tảng mà họ đã xây dựng. Để giải quyết [khủng hoảng](https://en.wikipedia.org/wiki/Foundations_of_mathematics#Foundational_crisis) đó, họ quay lại điểm xuất phát. Bắt đầu từ một vài tiên đề, logic và lý thuyết tập hợp, họ hy vọng có thể xây dựng lại toán học trên một nền móng không thể bị phá vỡ.

<aside name="paradox">

Nổi tiếng nhất là [**nghịch lý Russell**](https://en.wikipedia.org/wiki/Russell%27s_paradox). Ban đầu, lý thuyết tập hợp cho phép bạn định nghĩa bất kỳ loại tập hợp nào. Nếu bạn có thể mô tả nó bằng tiếng Anh, thì nó hợp lệ. Tất nhiên, với sở thích tự tham chiếu của các nhà toán học, các tập hợp có thể chứa những tập hợp khác. Thế là Russell — tay “láu cá” ấy — đã nghĩ ra:

*R là tập hợp của tất cả các tập hợp không chứa chính nó.*

R có chứa chính nó không? Nếu không, thì theo nửa sau của định nghĩa, nó phải chứa chính nó. Nhưng nếu có, thì nó lại không còn thỏa định nghĩa nữa. Xin mời… nổ não.

</aside>

Họ muốn trả lời một cách chặt chẽ những câu hỏi như: “Liệu tất cả các mệnh đề đúng có thể được chứng minh không?”, “Chúng ta có thể [tính toán](https://en.wikipedia.org/wiki/Computable_function) được tất cả các hàm mà ta có thể định nghĩa không?”, hay thậm chí là câu hỏi tổng quát hơn: “Chúng ta thực sự có ý gì khi nói một hàm là ‘computable’?”

Họ cho rằng câu trả lời cho hai câu hỏi đầu tiên sẽ là “có”. Việc còn lại chỉ là chứng minh nó. Nhưng hóa ra câu trả lời cho cả hai lại là “không”, và đáng ngạc nhiên là hai câu hỏi này lại gắn bó chặt chẽ với nhau. Đây là một góc thú vị của toán học, chạm đến những câu hỏi nền tảng về khả năng của bộ não và cách vũ trụ vận hành. Tôi không thể trình bày đầy đủ ở đây.

Điều tôi muốn nhấn mạnh là: trong quá trình chứng minh câu trả lời cho hai câu hỏi đầu tiên là “không”, Alan Turing và Alonzo Church đã đưa ra một câu trả lời chính xác cho câu hỏi cuối cùng — một định nghĩa về những loại hàm nào là <span name="uncomputable">computable</span>. Mỗi người đã tạo ra một hệ thống tí hon với lượng “máy móc” tối thiểu nhưng vẫn đủ mạnh để tính toán bất kỳ hàm nào trong một lớp (rất) lớn các hàm.

<aside name="uncomputable">

Họ chứng minh câu trả lời cho câu hỏi đầu tiên là “không” bằng cách chỉ ra rằng hàm trả về giá trị đúng/sai của một mệnh đề bất kỳ *không* phải là một hàm computable.

</aside>

Ngày nay, đây được xem là các “hàm computable”. Hệ thống của Turing được gọi là <span name="turing">**máy Turing**</span>. Hệ thống của Church là **lambda calculus**. Cả hai vẫn được dùng rộng rãi làm nền tảng cho các mô hình tính toán, và thực tế là nhiều ngôn ngữ lập trình hàm hiện đại sử dụng lambda calculus làm lõi.

<aside name="turing">

Turing gọi phát minh của mình là “a-machine” (automatic machine). Ông không tự đề cao đến mức đặt tên mình vào đó. Sau này, các nhà toán học khác đã làm điều đó thay ông. Đó là cách bạn trở nên nổi tiếng mà vẫn giữ được chút khiêm tốn.

</aside>

<img src="image/control-flow/turing-machine.png" alt="A Turing machine." />

Máy Turing nổi tiếng hơn — chưa có bộ phim Hollywood nào về Alonzo Church — nhưng hai hệ thống này [tương đương về sức mạnh](https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis). Thực tế, bất kỳ ngôn ngữ lập trình nào có mức độ biểu đạt tối thiểu cũng đủ mạnh để tính toán *bất kỳ* hàm computable nào.

Bạn có thể chứng minh điều đó bằng cách viết một trình mô phỏng máy Turing trong ngôn ngữ của mình. Vì Turing đã chứng minh máy của ông có thể tính toán bất kỳ hàm computable nào, nên theo đó, ngôn ngữ của bạn cũng có thể. Tất cả những gì bạn cần làm là dịch hàm đó sang máy Turing, rồi chạy nó trên trình mô phỏng của bạn.

Nếu ngôn ngữ của bạn đủ biểu đạt để làm điều đó, nó được xem là **Turing-complete**. Máy Turing thực ra khá đơn giản, nên không cần nhiều sức mạnh để đạt được điều này. Bạn chỉ cần có phép toán số học, một chút control flow, và khả năng cấp phát và sử dụng (về lý thuyết) lượng bộ nhớ tùy ý. Chúng ta đã có điều đầu tiên. Đến cuối chương này, chúng ta sẽ có <span name="memory">điều thứ hai</span>.

<aside name="memory">

Chúng ta *gần như* đã có điều thứ ba. Bạn có thể tạo và nối chuỗi với kích thước tùy ý, nên bạn *có thể* lưu trữ bộ nhớ không giới hạn. Nhưng chúng ta chưa có cách nào để truy cập một phần của chuỗi.

</aside>

## Conditional Execution (Execute có điều kiện)

Đủ với phần lịch sử rồi, giờ hãy làm cho ngôn ngữ của chúng ta thú vị hơn. Ta có thể chia control flow thành hai loại chính:

*   **Conditional** hay **branching control flow** được dùng để *không* execute một đoạn code nào đó. Theo cách nhìn của lập trình mệnh lệnh, bạn có thể hình dung nó như việc “nhảy *vượt qua*” một vùng code.

*   **Looping control flow** execute một đoạn code nhiều hơn một lần. Nó “nhảy *lùi lại*” để bạn có thể làm lại điều gì đó. Vì bạn thường không muốn vòng lặp *vô hạn*, nó thường đi kèm một logic điều kiện để biết khi nào cần dừng.

Branching đơn giản hơn, nên ta sẽ bắt đầu từ đây. Các ngôn ngữ họ C có hai tính năng chính để execute có điều kiện: câu lệnh `if` và toán tử “conditional” <span name="ternary">được đặt tên rất rõ nghĩa</span> (`?:`). Câu lệnh `if` cho phép bạn execute có điều kiện các statement, còn toán tử conditional cho phép bạn execute có điều kiện các expression.

<aside name="ternary">

Toán tử conditional còn được gọi là toán tử “ternary” vì nó là toán tử duy nhất trong C nhận ba toán hạng.

</aside>

Để đơn giản, Lox không có toán tử conditional, nên hãy bắt đầu với câu lệnh `if`. Grammar cho statement của chúng ta sẽ có thêm một production mới.

<span name="semicolon"></span>

```ebnf
statement      → exprStmt
               | ifStmt
               | printStmt
               | block ;

ifStmt         → "if" "(" expression ")" statement
               ( "else" statement )? ;
```

<aside name="semicolon">

Dấu chấm phẩy trong các rule không được đặt trong dấu nháy, nghĩa là chúng là một phần của metasyntax của grammar, không phải syntax của Lox. Một block không có `;` ở cuối và một câu lệnh `if` cũng vậy, trừ khi then hoặc else statement tình flag là một statement kết thúc bằng dấu chấm phẩy.

</aside>

Một câu lệnh `if` có một expression làm điều kiện, sau đó là một statement sẽ được execute nếu điều kiện là truthy. Tùy chọn, nó cũng có thể có từ khóa `else` và một statement sẽ được execute nếu điều kiện là falsey. <span name="if-ast">Node của syntax tree</span> sẽ có các field cho cả ba phần này.

^code if-ast (1 before, 1 after)

<aside name="if-ast">

Code được generated cho node mới này nằm trong [Appendix II](appendix-ii.html#if-statement).

</aside>

Giống như các statement khác, parser nhận biết một câu lệnh `if` bằng từ khóa `if` ở đầu.

^code match-if (1 before, 1 after)

Khi gặp nó, parser sẽ gọi method mới này để parse phần còn lại:

^code if-statement

<aside name="parens">

Dấu ngoặc đơn quanh điều kiện chỉ hữu ích một nửa. Bạn cần một loại ký hiệu phân tách *giữa* điều kiện và then statement, nếu không parser sẽ không biết khi nào điều kiện kết thúc. Nhưng dấu ngoặc *mở* ngay sau `if` thì không thực sự hữu ích. Dennis Ritchie đặt nó ở đó để ông có thể dùng `)` làm ký hiệu kết thúc mà không bị mất cân bằng ngoặc.

Các ngôn ngữ khác như Lua và một số BASIC dùng từ khóa như `then` làm ký hiệu kết thúc và không có gì trước điều kiện. Go và Swift thì yêu cầu statement phải là một block có ngoặc nhọn. Điều đó cho phép chúng dùng `{` ở đầu statement để xác định khi nào điều kiện kết thúc.

</aside>

Như thường lệ, code parse bám sát grammar. Nó phát hiện một else clause bằng cách tìm từ khóa `else` ngay trước đó. Nếu không có, field `elseBranch` trong syntax tree sẽ là `null`.

Tùy chọn else tưởng chừng vô hại này thực ra đã mở ra một điểm mơ hồ trong grammar. Hãy xem ví dụ:

```lox
if (first) if (second) whenTrue(); else whenFalse();
```

Câu đố là: else clause đó thuộc về câu lệnh `if` nào? Đây không chỉ là câu hỏi lý thuyết về cách ta ghi grammar, mà nó thực sự ảnh hưởng đến cách code chạy:

*   Nếu ta gắn else vào câu lệnh `if` đầu tiên, thì `whenFalse()` sẽ được gọi nếu `first` là falsey, bất kể `second` có giá trị gì.

*   Nếu ta gắn nó vào câu lệnh `if` thứ hai, thì `whenFalse()` chỉ được gọi nếu `first` là truthy và `second` là falsey.

Vì else clause là tùy chọn, và không có ký hiệu phân tách rõ ràng đánh dấu kết thúc câu lệnh `if`, grammar sẽ trở nên mơ hồ khi bạn lồng các `if` theo cách này. Đây là một cạm bẫy kinh điển của cú pháp, được gọi là vấn đề **[dangling else](https://en.wikipedia.org/wiki/Dangling_else)**.

<span name="else"></span>

<img class="above" src="image/control-flow/dangling-else.png" alt="Hai cách mà else có thể được diễn giải." />

<aside name="else">

Ở đây, cách format làm nổi bật hai cách mà else có thể được parse. Nhưng lưu ý rằng vì parser bỏ qua các ký tự whitespace, đây chỉ là gợi ý cho người đọc.

</aside>

Hoàn toàn *có thể* định nghĩa một context-free grammar để tránh sự mơ hồ này một cách trực tiếp, nhưng điều đó đòi hỏi phải tách hầu hết các rule cho statement thành từng cặp: một loại cho phép `if` có `else` và một loại không. Khá là phiền phức.

Thay vào đó, hầu hết các ngôn ngữ và parser đều tránh vấn đề này theo cách ad hoc. Dù họ dùng “mẹo” gì để thoát khỏi rắc rối, họ luôn chọn cùng một cách diễn giải — `else` sẽ gắn với câu lệnh `if` gần nhất đứng trước nó.

Parser của chúng ta thật tiện lợi khi đã làm đúng như vậy. Vì `ifStatement()` luôn chủ động tìm `else` trước khi trả về, nên lời gọi sâu nhất trong chuỗi `if` lồng nhau sẽ “giành” else clause cho mình trước khi trả quyền điều khiển về các câu lệnh `if` bên ngoài.

Giờ đã có cú pháp trong tay, chúng ta sẵn sàng để interpret.

^code visit-if

Phần implement trong interpreter chỉ là một lớp bọc mỏng quanh chính đoạn code Java tương ứng. Nó sẽ evaluate điều kiện. Nếu điều kiện là truthy, nó execute then branch. Ngược lại, nếu có else branch, nó sẽ execute phần đó.

Nếu bạn so sánh đoạn code này với cách interpreter xử lý các cú pháp khác mà ta đã implement, điểm khiến control flow trở nên đặc biệt chính là câu lệnh `if` của Java. Hầu hết các syntax tree khác luôn evaluate toàn bộ subtree của chúng. Ở đây, ta có thể *không* evaluate then hoặc else statement. Nếu một trong hai có side effect, việc không evaluate nó sẽ trở nên “thấy được” đối với người dùng.

## Logical Operators

Vì chúng ta không có toán tử conditional, bạn có thể nghĩ rằng phần branching đã xong, nhưng chưa đâu. Ngay cả khi không có toán tử ternary, vẫn còn hai toán tử khác về mặt kỹ thuật cũng là control flow construct — đó là các logical operator `and` và `or`.

Chúng không giống các binary operator khác vì chúng **short-circuit**. Nếu sau khi evaluate toán hạng bên trái mà ta đã biết kết quả của biểu thức logic sẽ là gì, thì ta sẽ không evaluate toán hạng bên phải nữa. Ví dụ:

```lox
false and sideEffect();
```

Với một biểu thức `and` để ra kết quả truthy, cả hai toán hạng phải truthy. Ngay khi evaluate toán hạng bên trái là `false`, ta biết điều kiện đó không thể xảy ra, nên không cần evaluate `sideEffect()` và nó sẽ bị bỏ qua.

Đây là lý do tại sao ta không implement logical operator cùng với các binary operator khác. Giờ thì ta đã sẵn sàng. Hai toán tử mới này nằm khá thấp trong bảng precedence. Tương tự như `||` và `&&` trong C, mỗi toán tử có <span name="logical">precedence riêng</span>, với `or` thấp hơn `and`. Ta sẽ đặt chúng ngay giữa `assignment` và `equality`.

<aside name="logical">

Tôi luôn tự hỏi tại sao chúng không có cùng precedence, giống như các toán tử so sánh hoặc equality khác.

</aside>

```ebnf
expression     → assignment ;
assignment     → IDENTIFIER "=" assignment
               | logic_or ;
logic_or       → logic_and ( "or" logic_and )* ;
logic_and      → equality ( "and" equality )* ;
```

Thay vì quay về `equality`, giờ `assignment` sẽ chuyển tiếp sang `logic_or`. Hai rule mới, `logic_or` và `logic_and`, <span name="same">tương tự</span> như các binary operator khác. Sau đó, `logic_and` sẽ gọi `equality` cho các toán hạng của nó, và từ đó ta nối lại với phần còn lại của các rule expression.

<aside name="same">

*Syntax* không quan tâm đến việc chúng short-circuit. Đó là vấn đề thuộc về semantics.

</aside>

Chúng ta *có thể* tái sử dụng class `Expr.Binary` hiện có cho hai expression mới này vì chúng có cùng các field. Nhưng khi đó, `visitBinaryExpr()` sẽ phải kiểm tra xem operator có phải là logical operator không và dùng một nhánh code khác để xử lý short-circuit. Tôi nghĩ sẽ gọn gàng hơn nếu định nghĩa một <span name="logical-ast">class mới</span> cho các toán tử này để chúng có visit method riêng.

^code logical-ast (1 before, 1 after)

<aside name="logical-ast">

Code được generated cho node mới này nằm trong [Appendix II](appendix-ii.html#logical-expression).

</aside>

Để đưa các expression mới này vào parser, trước tiên ta thay đổi code parse cho assignment để gọi `or()`.

^code or-in-assignment (1 before, 2 after)

Code để parse một chuỗi các biểu thức `or` giống với các binary operator khác.

^code or

Các toán hạng của nó sẽ là cấp precedence cao hơn tiếp theo — biểu thức `and` mới.

^code and

Biểu thức này sẽ gọi `equality()` cho các toán hạng của nó, và như vậy, parser cho expression đã được nối kết đầy đủ trở lại. Giờ ta sẵn sàng để interpret.

^code visit-logical

Nếu bạn so sánh đoạn này với method `visitBinaryExpr()` trong [chương trước](evaluating-expressions.html), bạn sẽ thấy sự khác biệt. Ở đây, chúng ta evaluate toán hạng bên trái trước. Ta xem giá trị của nó để quyết định có thể short-circuit hay không. Nếu không, *và chỉ khi đó*, ta mới evaluate toán hạng bên phải.

Điểm thú vị khác ở đây là quyết định giá trị thực sự sẽ trả về. Vì Lox là ngôn ngữ dynamically typed, chúng ta cho phép toán hạng thuộc bất kỳ kiểu nào và dùng truthiness để xác định ý nghĩa của mỗi toán hạng. Ta áp dụng cùng cách suy luận đó cho kết quả. Thay vì hứa sẽ trả về đúng `true` hoặc `false`, một logic operator chỉ đảm bảo rằng nó sẽ trả về một giá trị có truthiness phù hợp.

May mắn thay, chúng ta đã có sẵn các giá trị với truthiness đúng — chính là kết quả của các toán hạng. Vậy nên ta dùng luôn chúng. Ví dụ:

```lox
print "hi" or 2; // "hi".
print nil or "yes"; // "yes".
```

Ở dòng đầu tiên, `"hi"` là truthy, nên `or` short-circuit và trả về giá trị đó. Ở dòng thứ hai, `nil` là falsey, nên nó evaluate và trả về toán hạng thứ hai, `"yes"`.

Vậy là chúng ta đã bao quát hết các primitive branching trong Lox. Giờ thì sẵn sàng “nhảy” sang vòng lặp. Thấy tôi chơi chữ chứ? *Jump. Ahead.* Hiểu không? Ý là… thôi, bỏ qua đi.

## While Loops

Lox có hai câu lệnh looping control flow: `while` và `for`. Vòng lặp `while` đơn giản hơn, nên ta sẽ bắt đầu từ đây. Grammar của nó giống hệt C.

```ebnf
statement      → exprStmt
               | ifStmt
               | printStmt
               | whileStmt
               | block ;

whileStmt      → "while" "(" expression ")" statement ;
```

Ta thêm một nhánh mới vào rule `statement` trỏ đến rule mới cho `while`. Nó bắt đầu bằng từ khóa `while`, theo sau là một biểu thức điều kiện đặt trong ngoặc đơn, rồi đến một statement cho phần thân vòng lặp. Rule grammar mới này sẽ có một <span name="while-ast">node trong syntax tree</span>.

^code while-ast (1 before, 1 after)

<aside name="while-ast">

Code được generated cho node mới này nằm trong [Appendix II](appendix-ii.html#while-statement).

</aside>

Node này lưu trữ điều kiện và phần thân. Ở đây bạn có thể thấy lý do tại sao việc tách riêng base class cho expression và statement lại hữu ích. Các khai báo field cho thấy rõ ràng điều kiện là một expression và phần thân là một statement.

Bên phía parser, ta làm theo đúng quy trình đã dùng cho câu lệnh `if`. Đầu tiên, ta thêm một case mới trong `statement()` để phát hiện và khớp từ khóa ở đầu.

^code match-while (1 before, 1 after)

Việc xử lý thực sự được giao cho method này:

^code while-statement

Grammar này cực kỳ đơn giản và đây là bản dịch thẳng của nó sang Java. Nói đến dịch thẳng sang Java, đây là cách ta execute cú pháp mới:

^code visit-while

Giống như visit method cho `if`, visitor này dùng chính tính năng tương ứng của Java. Method này không phức tạp, nhưng nó khiến Lox mạnh mẽ hơn nhiều. Giờ ta có thể viết một chương trình mà thời gian chạy không còn bị giới hạn nghiêm ngặt bởi độ dài của source code.

## For Loops

Chúng ta đã đến control flow construct cuối cùng, <span name="for">Ye Olde</span> vòng lặp `for` kiểu C. Có lẽ tôi không cần nhắc bạn, nhưng nó trông như thế này:

```lox
for (var i = 0; i < 10; i = i + 1) print i;
```

Theo ngôn ngữ grammar, nó là:

```ebnf
statement      → exprStmt
               | forStmt
               | ifStmt
               | printStmt
               | whileStmt
               | block ;

forStmt        → "for" "(" ( varDecl | exprStmt | ";" )
                 expression? ";"
                 expression? ")" statement ;
```

<aside name="for">

Hầu hết các ngôn ngữ hiện đại đều có câu lệnh vòng lặp cấp cao hơn để duyệt qua các sequence do người dùng định nghĩa. C# có `foreach`, Java có “enhanced for”, thậm chí C++ giờ cũng có `for` dựa trên range. Chúng mang lại cú pháp gọn gàng hơn so với `for` của C bằng cách ngầm gọi vào một iteration protocol mà đối tượng được lặp hỗ trợ.

Tôi rất thích những thứ đó. Nhưng với Lox, chúng ta bị giới hạn bởi việc xây dựng interpreter từng chương một. Chúng ta chưa có object và method, nên chưa thể định nghĩa một iteration protocol mà vòng lặp `for` có thể dùng. Vậy nên ta sẽ gắn bó với vòng lặp `for` kiểu C cổ điển. Hãy coi nó như một món “vintage” — chiếc fixie của các câu lệnh control flow.

</aside>

Bên trong dấu ngoặc đơn, bạn có ba mệnh đề được phân tách bằng dấu chấm phẩy:

1.  Mệnh đề đầu tiên là *initializer*. Nó được execute đúng một lần, trước mọi thứ khác. Thông thường nó là một expression, nhưng để tiện lợi, chúng ta cũng cho phép một khai báo biến. Trong trường hợp đó, biến sẽ có scope trong phần còn lại của vòng lặp `for` — tức hai mệnh đề còn lại và phần thân.

2.  Tiếp theo là *condition*. Giống như trong vòng lặp `while`, expression này kiểm soát thời điểm thoát vòng lặp. Nó được evaluate một lần ở đầu mỗi lượt lặp, bao gồm cả lượt đầu tiên. Nếu kết quả là truthy, nó execute phần thân vòng lặp. Ngược lại, nó sẽ thoát.

3.  Mệnh đề cuối cùng là *increment*. Đây là một expression tùy ý làm một việc nào đó ở cuối mỗi lượt lặp. Kết quả của expression sẽ bị bỏ đi, vì thế để có ích thì nó phải có side effect. Trên thực tế, nó thường tăng giá trị của một biến.

Bất kỳ mệnh đề nào trong số này cũng có thể được lược bỏ. Sau dấu ngoặc đơn đóng là một statement cho phần thân, thường là một block.

### Desugaring

Khá nhiều “máy móc”, nhưng lưu ý rằng không có phần nào làm những việc mà bạn không thể làm với các câu lệnh chúng ta đã có. Nếu vòng lặp `for` không hỗ trợ mệnh đề initializer, bạn có thể đặt expression initializer trước câu lệnh `for`. Không có mệnh đề increment, bạn chỉ việc đặt expression increment ở cuối phần thân.

Nói cách khác, Lox không *cần* vòng lặp `for`, chỉ là nó khiến một số mẫu code thường gặp trở nên dễ viết hơn. Những tính năng kiểu này được gọi là <span name="sugar">**syntactic sugar**</span>. Ví dụ, vòng lặp `for` trước đó có thể được viết lại như sau:

<aside name="sugar">

Cụm từ duyên dáng này do Peter J. Landin đặt ra năm 1964 để mô tả cách một số dạng biểu thức đẹp đẽ được các ngôn ngữ như ALGOL hỗ trợ thực chất là lớp “đường” rắc lên trên phần nền tảng — nhưng có lẽ kém ngon miệng hơn — là lambda calculus bên dưới.

<img class="above" src="image/control-flow/sugar.png" alt="Slightly more than a spoonful of sugar." />

</aside>

```lox
{
  var i = 0;
  while (i < 10) {
    print i;
    i = i + 1;
  }
}
```

Script này có semantics giống hệt với script trước đó, dù nhìn không “mát mắt” bằng. Những tính năng syntactic sugar như vòng lặp `for` của Lox khiến ngôn ngữ dễ chịu và hiệu quả hơn khi làm việc. Nhưng đặc biệt trong các implementation ngôn ngữ tinh vi, mỗi tính năng cần hỗ trợ và tối ưu ở back-end đều tốn kém.

Chúng ta có thể “vừa ăn bánh vừa giữ bánh” bằng cách <span
name="caramel">**desugaring**</span>. Từ ngữ vui tai này mô tả quá trình trong đó front end nhận code dùng cú pháp sugar và dịch nó sang một dạng nguyên thủy hơn mà back end đã biết cách execute.

<aside name="caramel">

Ôi, tôi ước gì thuật ngữ được chấp nhận là “caramelization”. Đã đưa ra ẩn dụ thì sao lại không đi đến cùng nhỉ?

</aside>

Chúng ta sẽ desugar vòng lặp `for` thành vòng lặp `while` và các câu lệnh khác mà interpreter đã xử lý được. Trong interpreter đơn giản của chúng ta, desugaring thật ra không tiết kiệm được bao nhiêu công, nhưng nó cho tôi cái cớ để giới thiệu bạn với kỹ thuật này. Vì vậy, khác với các câu lệnh trước, chúng ta *sẽ không* thêm một node mới vào syntax tree. Thay vào đó, ta đi thẳng vào parsing. Đầu tiên, thêm một import mà ta sắp cần.

^code import-arrays (1 before, 1 after)

Giống như mọi câu lệnh khác, ta bắt đầu parse một vòng lặp `for` bằng cách khớp từ khóa của nó.

^code match-for (1 before, 1 after)

Giờ thì thú vị đây. Việc desugaring sẽ diễn ra ở đây, nên ta sẽ xây dựng method này từng phần, bắt đầu bằng dấu ngoặc mở trước các mệnh đề.

^code for-statement

Mệnh đề đầu tiên sau đó là initializer.

^code for-initializer (2 before, 1 after)

Nếu token ngay sau `(` là dấu chấm phẩy thì initializer đã bị lược bỏ. Ngược lại, ta kiểm tra từ khóa `var` để xem đó có phải là một khai báo <span
name="variable">variable</span> hay không. Nếu không rơi vào trường hợp nào, chắc chắn đó là một expression. Ta parse nó và bọc nó trong một expression statement để initializer luôn có kiểu Stmt.

<aside name="variable">

Ở một chương trước, tôi đã nói rằng ta có thể tách syntax tree cho expression và statement thành hai hệ phân cấp class riêng biệt vì không có một vị trí nào trong grammar cho phép cả expression lẫn statement. Có lẽ điều đó không *hoàn toàn* đúng, nhỉ.

</aside>

Tiếp theo là phần điều kiện.

^code for-condition (2 before, 1 after)

Một lần nữa, ta kiểm tra dấu chấm phẩy để xem mệnh đề này có bị lược bỏ hay không. Mệnh đề cuối cùng là phần increment.

^code for-increment (1 before, 1 after)

Nó giống với mệnh đề điều kiện, chỉ khác là mệnh đề này kết thúc bằng dấu ngoặc đơn đóng. Phần còn lại chỉ là <span name="body">body</span>.

<aside name="body">

Chỉ mình tôi thấy hay nó nghe hơi… rùng rợn? “Tất cả những gì còn lại… là *cái xác*”.

</aside>

^code for-body (1 before, 1 after)

Chúng ta đã parse xong tất cả các phần của vòng lặp `for` và các AST node kết quả đang nằm trong một vài biến local của Java. Đây là lúc desugaring xuất hiện. Ta lấy chúng và dùng để tạo ra các node của syntax tree thể hiện semantics của vòng lặp `for`, giống như ví dụ “tự desugar bằng tay” mà tôi đã cho bạn xem trước đó.

Code sẽ đơn giản hơn một chút nếu ta làm ngược lại, nên ta bắt đầu với mệnh đề increment.

^code for-desugar-increment (2 before, 1 after)

Phần increment, nếu có, sẽ chạy sau body ở mỗi vòng lặp. Ta làm điều đó bằng cách thay thế body bằng một block nhỏ chứa body gốc, sau đó là một expression statement để evaluate phần increment.

^code for-desugar-condition (2 before, 1 after)

Tiếp theo, ta lấy điều kiện và body để tạo vòng lặp bằng `while` loop nguyên thủy. Nếu điều kiện bị lược bỏ, ta chèn `true` để tạo vòng lặp vô hạn.

^code for-desugar-initializer (2 before, 1 after)

Cuối cùng, nếu có initializer, nó sẽ chạy một lần trước toàn bộ vòng lặp. Ta làm điều đó bằng cách, một lần nữa, thay thế toàn bộ statement bằng một block chạy initializer rồi execute vòng lặp.

Vậy là xong. Interpreter của chúng ta giờ đã hỗ trợ vòng lặp `for` kiểu C mà không cần đụng gì đến class Interpreter. Vì ta đã desugar thành các node mà interpreter vốn đã biết cách visit, nên không còn việc gì phải làm thêm.

Cuối cùng, Lox đã đủ mạnh để giải trí cho chúng ta, ít nhất là vài phút. Đây là một chương trình nhỏ in ra 21 phần tử đầu tiên của dãy Fibonacci:

```lox
var a = 0;
var temp;

for (var b = 1; a < 10000; b = temp + b) {
  print a;
  temp = a;
  a = b;
}
```

<div class="challenges">

## Challenges

1.  Vài chương nữa, khi Lox hỗ trợ first-class function và dynamic dispatch, về mặt kỹ thuật chúng ta sẽ không *cần* các câu lệnh branching được tích hợp sẵn trong ngôn ngữ. Hãy chỉ ra cách execute có điều kiện có thể được triển khai dựa trên những tính năng đó. Nêu tên một ngôn ngữ sử dụng kỹ thuật này cho control flow của nó.

2.  Tương tự, looping cũng có thể được triển khai bằng những công cụ đó, miễn là interpreter của chúng ta hỗ trợ một tối ưu hóa quan trọng. Đó là gì, và tại sao nó lại cần thiết? Nêu tên một ngôn ngữ sử dụng kỹ thuật này cho iteration.

3.  Không giống Lox, hầu hết các ngôn ngữ kiểu C khác cũng hỗ trợ câu lệnh `break` và `continue` bên trong vòng lặp. Hãy thêm hỗ trợ cho câu lệnh `break`.

    Cú pháp là từ khóa `break` theo sau bởi dấu chấm phẩy. Sẽ là lỗi cú pháp nếu có câu lệnh `break` xuất hiện bên ngoài bất kỳ vòng lặp bao quanh nào. Khi runtime gặp câu lệnh `break`, nó sẽ nhảy đến cuối vòng lặp bao quanh gần nhất và tiếp tục từ đó. Lưu ý rằng `break` có thể nằm lồng bên trong các block hoặc câu lệnh `if` khác cũng cần được thoát ra.

</div>

<div class="design-note">

## Design Note: Spoonfuls of Syntactic Sugar

Khi bạn thiết kế ngôn ngữ của riêng mình, bạn sẽ chọn lượng syntactic sugar để “rót” vào grammar. Bạn sẽ tạo ra một món ăn “healthy” không đường, nơi mỗi thao tác semantics ánh xạ 1-1 với một đơn vị cú pháp, hay một món tráng miệng ngọt ngào, nơi mỗi hành vi có thể được diễn đạt theo mười cách khác nhau? Các ngôn ngữ thành công tồn tại ở mọi điểm trên phổ này.

Ở cực tối giản là những ngôn ngữ có cú pháp tối thiểu đến mức khắc nghiệt như Lisp, Forth và Smalltalk. Dân Lisp nổi tiếng với câu “ngôn ngữ của họ không có cú pháp”, trong khi dân Smalltalk tự hào cho thấy bạn có thể nhét toàn bộ grammar lên một tấm thẻ ghi chú. Trường phái này tin rằng *ngôn ngữ* không cần syntactic sugar. Thay vào đó, cú pháp và semantics tối giản mà nó cung cấp đủ mạnh để code trong thư viện có thể biểu đạt như thể nó là một phần của ngôn ngữ.

Gần đó là các ngôn ngữ như C, Lua và Go. Chúng hướng tới sự đơn giản và rõ ràng hơn là tối giản tuyệt đối. Một số, như Go, cố tình tránh cả syntactic sugar lẫn khả năng mở rộng cú pháp như nhóm trước. Chúng muốn cú pháp không cản trở semantics, nên tập trung giữ cho cả grammar và thư viện đơn giản. Code nên rõ ràng hơn là đẹp đẽ.

Ở khoảng giữa, bạn có các ngôn ngữ như Java, C# và Python. Rồi dần dần bạn sẽ gặp Ruby, C++, Perl và D — những ngôn ngữ nhồi nhét quá nhiều cú pháp vào grammar đến mức gần như hết ký tự dấu câu trên bàn phím.

Ở một mức độ nào đó, vị trí trên phổ này có liên quan đến tuổi đời. Việc thêm chút syntactic sugar ở các bản phát hành sau tương đối dễ dàng. Cú pháp mới thường được đón nhận, và ít có khả năng phá vỡ chương trình hiện có hơn so với việc thay đổi semantics. Một khi đã thêm vào, bạn không thể gỡ bỏ, nên các ngôn ngữ có xu hướng “ngọt” dần theo thời gian. Một trong những lợi ích lớn của việc tạo ra một ngôn ngữ mới từ đầu là bạn có cơ hội cạo bỏ những lớp “kem” tích tụ và bắt đầu lại.

Syntactic sugar thường bị giới PL intelligentsia đánh giá thấp. Có một “niềm đam mê” thực sự với chủ nghĩa tối giản trong nhóm này. Điều đó cũng có lý do: cú pháp được thiết kế kém, không cần thiết sẽ làm tăng gánh nặng nhận thức mà không thêm đủ khả năng biểu đạt để xứng đáng. Vì luôn có áp lực nhồi nhét tính năng mới vào ngôn ngữ, cần có kỷ luật và tập trung vào sự đơn giản để tránh phình to. Một khi đã thêm cú pháp, bạn sẽ phải sống chung với nó, nên tốt nhất là tiết kiệm.

Đồng thời, hầu hết các ngôn ngữ thành công đều có grammar khá phức tạp, ít nhất là khi chúng được sử dụng rộng rãi. Lập trình viên dành rất nhiều thời gian trong ngôn ngữ họ chọn, và một vài tiện ích nhỏ ở đây đó thực sự có thể cải thiện sự thoải mái và hiệu quả công việc.

Tìm được sự cân bằng hợp lý — chọn mức độ “ngọt” phù hợp cho ngôn ngữ của bạn — phụ thuộc vào gu thẩm mỹ của chính bạn.

</div>