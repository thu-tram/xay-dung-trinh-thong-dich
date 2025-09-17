> Cắn miếng to vào. Bất cứ việc gì đáng làm thì cũng đáng làm tới mức “quá tay”.
>
> <cite>Robert A. Heinlein, <em>Time Enough for Love</em></cite>

Bước đầu tiên trong bất kỳ compiler hay interpreter nào là <span name="lexing">scanning</span>. Scanner nhận vào mã nguồn thô dưới dạng một chuỗi ký tự và nhóm chúng thành một chuỗi các mảnh mà ta gọi là **token**. Đây là những “từ” và “dấu câu” mang ý nghĩa, tạo nên ngữ pháp của ngôn ngữ.

<aside name="lexing">

Nhiệm vụ này qua nhiều năm đã được gọi bằng nhiều tên khác nhau: “scanning” và “lexing” (viết tắt của “lexical analysis”). Ngày xưa, khi máy tính to bằng cả chiếc xe Winnebago nhưng bộ nhớ còn ít hơn cả đồng hồ của bạn, một số người chỉ dùng từ “scanner” để chỉ phần code xử lý việc đọc ký tự mã nguồn thô từ đĩa và đưa vào bộ nhớ đệm. Sau đó, “lexing” là giai đoạn tiếp theo, làm những việc hữu ích với các ký tự đó.

Ngày nay, việc đọc một file mã nguồn vào bộ nhớ là chuyện đơn giản, nên nó hiếm khi là một giai đoạn riêng biệt trong compiler. Vì vậy, hai thuật ngữ này giờ gần như có thể dùng thay thế cho nhau.

</aside>

Scanning cũng là một điểm khởi đầu tốt cho chúng ta vì phần code này không quá khó — về cơ bản là một câu lệnh `switch` nhưng “mơ mộng” hơn một chút. Nó sẽ giúp ta khởi động trước khi bước vào những phần thú vị hơn ở phía sau. Đến cuối chương này, ta sẽ có một scanner đầy đủ tính năng, chạy nhanh, có thể nhận bất kỳ chuỗi mã nguồn Lox nào và tạo ra các token để đưa vào parser trong chương tiếp theo.

## Khung Interpreter

Vì đây là chương thực sự đầu tiên, trước khi bắt tay vào quét code, ta cần phác thảo hình hài cơ bản của interpreter jlox. Mọi thứ bắt đầu với một class trong Java.

^code lox-class

<aside name="64">

Về exit code, tôi dùng quy ước được định nghĩa trong header UNIX
["sysexits.h"](https://www.freebsd.org/cgi/man.cgi?query=sysexits&amp;apropos=0&amp;sektion=0&amp;manpath=FreeBSD+4.3-RELEASE&amp;format=html). Đây là thứ gần giống tiêu chuẩn nhất mà tôi tìm được.

</aside>

Hãy lưu nó vào một file văn bản, rồi mở IDE hoặc Makefile hay bất cứ công cụ nào bạn dùng để thiết lập. Tôi sẽ ở đây chờ bạn sẵn sàng. Xong chứ? OK!

Lox là một ngôn ngữ scripting, nghĩa là nó execute trực tiếp từ mã nguồn. Interpreter của ta hỗ trợ hai cách chạy code. Nếu bạn khởi động jlox từ dòng lệnh và truyền vào đường dẫn tới một file, nó sẽ đọc file đó và execute.

^code run-file

Nếu bạn muốn “trò chuyện” gần gũi hơn với interpreter, bạn cũng có thể chạy nó ở chế độ tương tác. Khởi động jlox mà không truyền đối số nào, nó sẽ đưa bạn vào một prompt nơi bạn có thể nhập và execute code từng dòng một.

<aside name="repl">

Prompt tương tác còn được gọi là “REPL” (phát âm giống “rebel” nhưng với âm “p”). Tên gọi này xuất phát từ Lisp, nơi việc hiện thực một REPL đơn giản như việc bọc một vòng lặp quanh vài hàm dựng sẵn:

```lisp
(print (eval (read)))
```

Đi từ trong ra ngoài, bạn **R**ead (đọc) một dòng nhập, **E**valuate (đánh giá) nó, **P**rint (in) kết quả, rồi **L**oop (lặp) và làm lại từ đầu.

</aside>

^code prompt

Hàm `readLine()`, đúng như tên gọi, đọc một dòng nhập từ người dùng trên dòng lệnh và trả về kết quả. Để thoát một ứng dụng dòng lệnh tương tác, bạn thường nhấn Control-D. Thao tác này gửi tín hiệu “end-of-file” tới chương trình. Khi điều đó xảy ra, `readLine()` trả về `null`, nên ta kiểm tra điều này để thoát vòng lặp.

Cả prompt và trình chạy file đều là các lớp bọc mỏng quanh hàm lõi này:

^code run

Hiện tại nó chưa hữu ích lắm vì ta chưa viết interpreter, nhưng cứ từ từ từng bước một. Lúc này, nó sẽ in ra các token mà scanner sắp viết sẽ tạo ra, để ta có thể thấy mình đang tiến triển thế nào.

### Xử lý lỗi

Khi đang thiết lập mọi thứ, một phần hạ tầng quan trọng khác là *xử lý lỗi*. Sách giáo khoa đôi khi lướt qua phần này vì nó mang tính thực tiễn nhiều hơn là một vấn đề khoa học máy tính “hàn lâm”. Nhưng nếu bạn muốn tạo ra một ngôn ngữ thực sự *dùng được*, thì xử lý lỗi một cách mượt mà là điều sống còn.

Các công cụ mà ngôn ngữ của ta cung cấp để xử lý lỗi chiếm một phần lớn trong giao diện người dùng của nó. Khi code của người dùng chạy tốt, họ chẳng nghĩ gì về ngôn ngữ của ta cả — tâm trí họ hoàn toàn tập trung vào *chương trình của họ*. Thường thì chỉ khi mọi thứ trục trặc, họ mới để ý đến phần hiện thực của ta.

<span name="errors">Khi</span> điều đó xảy ra, nhiệm vụ của ta là cung cấp cho người dùng tất cả thông tin họ cần để hiểu chuyện gì đã sai và nhẹ nhàng dẫn họ quay lại đúng hướng. Làm tốt điều này nghĩa là phải nghĩ về xử lý lỗi xuyên suốt quá trình hiện thực interpreter, bắt đầu từ bây giờ.

<aside name="errors">

Nói vậy thôi, với interpreter *này*, những gì ta xây dựng sẽ khá đơn giản. Tôi rất muốn bàn về debugger tương tác, static analyzer, và những thứ thú vị khác, nhưng “mực” trong bút thì có hạn.

</aside>

^code lox-error

Hàm `error()` này và helper `report()` của nó sẽ báo cho người dùng biết có lỗi cú pháp xảy ra ở một dòng nhất định. Đây thực sự là mức tối thiểu để có thể nói rằng bạn *có* báo lỗi. Hãy tưởng tượng nếu bạn vô tình để sót một dấu phẩy trong lời gọi hàm và interpreter in ra:

```text
Error: Unexpected "," somewhere in your code. Good luck finding it!
```

Thế thì chẳng giúp ích gì mấy. Ta cần ít nhất chỉ ra đúng dòng. Tốt hơn nữa là chỉ ra cả cột bắt đầu và kết thúc để họ biết *chỗ nào* trong dòng. Còn tốt hơn *nữa* là *hiển thị* cho người dùng dòng code gây lỗi, như:

```text
Error: Unexpected "," in argument list.

    15 | function(first, second,);
                               ^-- Here.
```

Tôi rất muốn hiện thực thứ như vậy trong sách này, nhưng thật lòng mà nói thì nó đòi hỏi khá nhiều code xử lý chuỗi lắt nhắt. Rất hữu ích cho người dùng, nhưng không thú vị lắm để đọc trong sách và cũng không quá hấp dẫn về mặt kỹ thuật. Vậy nên ta sẽ chỉ dừng ở mức số dòng. Trong interpreter của riêng bạn, hãy làm như tôi khuyên, đừng như tôi làm ở đây.

Lý do chính mà ta đặt hàm báo lỗi này trong class `Lox` chính là vì trường `hadError`. Nó được định nghĩa ở đây:

^code had-error (1 before)

Ta sẽ dùng nó để đảm bảo không cố execute code đã biết là có lỗi. Ngoài ra, nó cho phép ta thoát với exit code khác 0 như một “công dân” dòng lệnh gương mẫu.

^code exit-code (1 before, 1 after)

Ta cần reset cờ này trong vòng lặp tương tác. Nếu người dùng mắc lỗi, nó không nên “giết” cả phiên làm việc của họ.

^code reset-had-error (1 before, 1 after)

Một lý do khác tôi tách phần báo lỗi ra đây thay vì nhét vào scanner hay các giai đoạn khác nơi lỗi có thể xảy ra là để nhắc bạn rằng việc tách biệt code *tạo* lỗi và code *báo* lỗi là một thực hành kỹ thuật tốt.

Nhiều giai đoạn của front-end sẽ phát hiện lỗi, nhưng không thực sự là nhiệm vụ của chúng để biết cách hiển thị lỗi cho người dùng. Trong một hiện thực ngôn ngữ đầy đủ tính năng, bạn có thể sẽ có nhiều cách hiển thị lỗi: trên stderr, trong cửa sổ lỗi của IDE, ghi vào file log, v.v. Bạn không muốn code đó bị rải khắp scanner và parser.

Lý tưởng nhất, ta sẽ có một abstraction thực sự, kiểu như một interface <span name="reporter">"ErrorReporter"</span> được truyền vào scanner và parser để có thể thay đổi chiến lược báo lỗi. Với interpreter đơn giản này, tôi không làm vậy, nhưng ít nhất tôi cũng đã chuyển code báo lỗi sang một class khác.

<aside name="reporter">

Khi lần đầu hiện thực jlox, tôi đã làm đúng như vậy. Nhưng cuối cùng tôi bỏ nó đi vì cảm thấy quá “over-engineered” so với interpreter tối giản trong sách này.

</aside>

Với một chút xử lý lỗi cơ bản, “vỏ” ứng dụng của ta đã sẵn sàng. Khi có class `Scanner` với phương thức `scanTokens()`, ta có thể bắt đầu chạy nó. Trước khi đến đó, hãy làm rõ hơn về token.

## Lexeme & Token

Đây là một dòng code Lox:

```lox
var language = "lox";
```

Ở đây, `var` là keyword để khai báo biến. Chuỗi ba ký tự “v-a-r” có ý nghĩa. Nhưng nếu ta lấy ba ký tự từ giữa `language`, như “g-u-a”, thì chúng chẳng có nghĩa gì cả.

Đó chính là điều mà phân tích từ vựng (lexical analysis) xử lý. Nhiệm vụ của ta là quét qua danh sách ký tự và nhóm chúng thành các chuỗi nhỏ nhất vẫn mang ý nghĩa. Mỗi “cục” ký tự như vậy được gọi là **lexeme**. Trong ví dụ trên, các lexeme là:

<img src="image/scanning/lexemes.png" alt="'var', 'language', '=', 'lox', ';'" />

Lexeme chỉ là các chuỗi con thô của mã nguồn. Tuy nhiên, trong quá trình nhóm các ký tự thành lexeme, ta cũng thu được một số thông tin hữu ích khác. Khi ta lấy lexeme và gói nó cùng dữ liệu đó, kết quả là một token. Nó bao gồm những thứ hữu ích như:

### Loại token (Token type)

Keyword là một phần của cấu trúc ngữ pháp ngôn ngữ, nên parser thường có code kiểu như: “Nếu token tiếp theo là `while` thì làm…”. Điều đó nghĩa là parser muốn biết không chỉ rằng nó có một lexeme cho một identifier nào đó, mà còn rằng nó là một từ khóa *reserved*, và *là từ khóa nào*.

<span name="ugly">Parser</span> có thể phân loại token từ lexeme thô bằng cách so sánh chuỗi, nhưng cách đó vừa chậm vừa xấu. Thay vào đó, ngay tại thời điểm ta nhận diện được một lexeme, ta cũng ghi nhớ *loại* lexeme mà nó biểu diễn. Ta có một loại riêng cho mỗi keyword, toán tử, dấu câu, và loại literal.

<aside name="ugly">

Xét cho cùng, so sánh chuỗi cũng phải duyệt từng ký tự, mà đó chẳng phải là việc của scanner sao?

</aside>

^code token-type

### Giá trị literal

Có những lexeme dành cho các giá trị literal — số, chuỗi và những thứ tương tự. Vì scanner phải duyệt qua từng ký tự trong literal để nhận diện chính xác, nó cũng có thể chuyển đổi biểu diễn dạng văn bản của giá trị đó thành đối tượng runtime “sống” sẽ được interpreter sử dụng sau này.

### Thông tin vị trí

Khi tôi nói về “phúc âm” của xử lý lỗi, ta đã thấy rằng ta cần cho người dùng biết *lỗi xảy ra ở đâu*. Việc theo dõi điều đó bắt đầu từ đây. Trong interpreter đơn giản của ta, ta chỉ ghi lại token xuất hiện ở dòng nào, nhưng các hiện thực phức tạp hơn sẽ bao gồm cả cột và độ dài.

<aside name="location">

Một số hiện thực token lưu thông tin vị trí dưới dạng hai số: offset từ đầu file nguồn tới đầu lexeme, và độ dài của lexeme. Scanner vốn dĩ cần biết những thông tin này, nên không tốn thêm chi phí để tính toán.

Offset có thể được chuyển đổi thành vị trí dòng và cột sau này bằng cách nhìn lại file nguồn và đếm số lần xuống dòng trước đó. Nghe có vẻ chậm, và đúng là vậy. Tuy nhiên, bạn chỉ cần làm điều đó *khi thực sự cần hiển thị dòng và cột cho người dùng*. Phần lớn token sẽ không bao giờ xuất hiện trong thông báo lỗi. Với những token đó, càng ít thời gian tính toán thông tin vị trí trước thì càng tốt.

</aside>

Ta gom tất cả dữ liệu này và gói nó trong một class.

^code token-class

Giờ ta đã có một đối tượng với đủ cấu trúc để hữu ích cho tất cả các giai đoạn sau của interpreter.

## Ngôn ngữ & biểu thức chính quy

Giờ ta đã biết mình muốn tạo ra cái gì, hãy… tạo ra nó. Lõi của scanner là một vòng lặp. Bắt đầu từ ký tự đầu tiên của mã nguồn, scanner xác định ký tự đó thuộc về lexeme nào, rồi tiêu thụ nó cùng bất kỳ ký tự tiếp theo nào thuộc về lexeme đó. Khi tới cuối lexeme, nó phát ra một token.

Sau đó, nó quay lại và làm lại từ đầu, bắt đầu từ ký tự tiếp theo trong mã nguồn. Nó cứ tiếp tục như vậy, “ăn” ký tự và thỉnh thoảng, ờ, “thải” ra token, cho tới khi tới cuối input.

<span name="alligator"></span>

<img src="image/scanning/lexigator.png" alt="Một con cá sấu đang ăn ký tự và, ừm, bạn không muốn biết đâu." />

<aside name="alligator">

Lexical analygator.

</aside>

Phần trong vòng lặp nơi ta nhìn vào một vài ký tự để xác định lexeme nó “khớp” có thể nghe quen thuộc. Nếu bạn biết regular expression, bạn có thể nghĩ tới việc định nghĩa một regex cho mỗi loại lexeme và dùng chúng để so khớp ký tự. Ví dụ, Lox có cùng quy tắc với C cho identifier (tên biến và tương tự). Regex này sẽ khớp một identifier:

```text
[a-zA-Z_][a-zA-Z_0-9]*
```

Nếu bạn nghĩ tới regular expression, thì trực giác của bạn rất sâu sắc. Các quy tắc xác định cách một ngôn ngữ nhóm ký tự thành lexeme được gọi là <span name="theory">**lexical grammar**</span> của nó. Trong Lox, cũng như hầu hết các ngôn ngữ lập trình, các quy tắc của ngữ pháp này đủ đơn giản để ngôn ngữ được xếp loại là **[regular language](https://en.wikipedia.org/wiki/Regular_language)**. Đây chính là “regular” trong regular expression.

<aside name="theory">

Tôi thấy tiếc khi phải lướt qua phần lý thuyết này, nhất là khi nó thú vị như [Chomsky hierarchy](https://en.wikipedia.org/wiki/Chomsky_hierarchy) và [finite-state machine](https://en.wikipedia.org/wiki/Finite-state_machine). Nhưng thật lòng mà nói, có những cuốn sách khác trình bày phần này tốt hơn tôi nhiều. [*Compilers: Principles, Techniques, and Tools*](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools) (được biết đến rộng rãi là “dragon book”) là tài liệu tham khảo kinh điển.

</aside>

Bạn hoàn toàn *có thể* nhận diện tất cả các lexeme khác nhau của Lox bằng regex nếu muốn, và có cả một đống lý thuyết thú vị giải thích tại sao và ý nghĩa của điều đó. Các công cụ như [Lex](http://dinosaur.compilertools.net/lex/) hoặc [Flex](https://github.com/westes/flex) được thiết kế riêng để làm việc này — bạn đưa cho chúng một loạt regex, và chúng sẽ trả lại cho bạn một scanner hoàn chỉnh.

<aside name="lex">

Lex được tạo ra bởi Mike Lesk và Eric Schmidt. Vâng, chính là Eric Schmidt từng là chủ tịch điều hành của Google. Tôi không nói rằng lập trình ngôn ngữ là con đường chắc chắn dẫn tới giàu sang và nổi tiếng, nhưng ta *có thể* kể ra ít nhất một tỷ phú trong giới này.

</aside>

Vì mục tiêu của ta là hiểu cách một scanner làm việc, ta sẽ không giao phó nhiệm vụ này. Chúng ta hướng tới những sản phẩm thủ công tinh xảo.

## Lớp Scanner

Không dài dòng nữa, hãy bắt tay vào tạo một scanner.

^code scanner-class

<aside name="static-import">

Tôi biết một số người cho rằng static import là phong cách xấu, nhưng nó giúp tôi khỏi phải rải `TokenType.` khắp scanner và parser. Hãy bỏ qua cho tôi, vì trong sách này từng ký tự đều đáng giá.

</aside>

Ta lưu mã nguồn thô dưới dạng một chuỗi đơn giản, và có sẵn một danh sách để chứa các token sẽ được tạo ra. Vòng lặp đã nhắc tới trước đó trông như thế này:

^code scan-tokens

Scanner sẽ duyệt qua mã nguồn, thêm token cho đến khi hết ký tự. Sau đó, nó thêm một token “end of file” cuối cùng. Thực ra thì không bắt buộc, nhưng nó giúp parser của ta gọn gàng hơn một chút.

Vòng lặp này dựa vào một vài trường để theo dõi vị trí hiện tại của scanner trong mã nguồn.

^code scan-state (1 before, 2 after)

Các trường `start` và `current` là các offset đánh chỉ số vào chuỗi. Trường `start` trỏ tới ký tự đầu tiên trong lexeme đang được quét, còn `current` trỏ tới ký tự hiện tại đang được xét. Trường `line` theo dõi `current` đang ở dòng nào trong mã nguồn để ta có thể tạo ra các token biết vị trí của mình.

Tiếp theo, ta có một hàm helper nhỏ để cho biết liệu ta đã tiêu thụ hết ký tự hay chưa.

^code is-at-end

## Nhận diện Lexeme

Mỗi vòng lặp, ta quét một token. Đây là phần cốt lõi thực sự của scanner. Ta sẽ bắt đầu đơn giản. Hãy tưởng tượng nếu mỗi lexeme chỉ dài một ký tự. Tất cả những gì cần làm là tiêu thụ ký tự tiếp theo và chọn loại token cho nó. Trong Lox, có một số lexeme *đúng là* chỉ một ký tự, nên hãy bắt đầu với chúng.

^code scan-token

<aside name="slash">

Bạn thắc mắc tại sao `/` chưa có ở đây? Đừng lo, ta sẽ xử lý nó sau.

</aside>

Một lần nữa, ta cần một vài phương thức helper.

^code advance-and-add-token

Phương thức `advance()` tiêu thụ ký tự tiếp theo trong file nguồn và trả về nó. Nếu `advance()` là để lấy input, thì `addToken()` là để xuất output. Nó lấy phần văn bản của lexeme hiện tại và tạo một token mới cho nó. Ta sẽ dùng overload khác để xử lý các token có giá trị literal sau.

### Lỗi từ vựng (Lexical errors)

Trước khi đi quá xa, hãy dành chút thời gian nghĩ về lỗi ở mức từ vựng. Điều gì xảy ra nếu người dùng đưa vào một file nguồn chứa các ký tự mà Lox không dùng, như `@#^`? Hiện tại, các ký tự đó sẽ bị bỏ qua trong im lặng. Chúng không được Lox sử dụng, nhưng điều đó không có nghĩa interpreter có thể giả vờ như chúng không tồn tại. Thay vào đó, ta sẽ báo lỗi.

^code char-error (1 before, 1 after)

Lưu ý rằng ký tự gây lỗi vẫn được *tiêu thụ* bởi lời gọi `advance()` trước đó. Điều này quan trọng để tránh bị kẹt trong vòng lặp vô hạn.

Cũng lưu ý rằng ta <span name="shotgun">*tiếp tục quét*</span>. Có thể còn những lỗi khác ở phần sau của chương trình. Việc phát hiện càng nhiều lỗi trong một lần chạy sẽ mang lại trải nghiệm tốt hơn cho người dùng. Nếu không, họ sẽ thấy một lỗi nhỏ, sửa nó, rồi lại thấy lỗi tiếp theo, cứ thế… Chơi trò “đập chuột” với lỗi cú pháp thì chẳng vui chút nào.

(Đừng lo. Vì `hadError` đã được đặt, ta sẽ không bao giờ cố *execute* bất kỳ code nào, dù vẫn tiếp tục quét phần còn lại.)

<aside name="shotgun">

Code sẽ báo từng ký tự không hợp lệ riêng lẻ, nên nếu người dùng lỡ dán một đống văn bản kỳ quặc, họ sẽ bị “xả” một loạt lỗi. Gom một chuỗi ký tự không hợp lệ thành một lỗi duy nhất sẽ mang lại trải nghiệm dễ chịu hơn.

</aside>

### Toán tử (Operators)

Ta đã xử lý xong các lexeme một ký tự, nhưng điều đó chưa bao quát hết các toán tử của Lox. Còn `!` thì sao? Nó là một ký tự đơn, đúng không? Đôi khi đúng, nhưng nếu ký tự ngay sau nó là dấu bằng, thì ta cần tạo lexeme `!=`. Lưu ý rằng `!` và `=` *không* phải là hai toán tử độc lập. Bạn không thể viết `!   =` trong Lox và mong nó hoạt động như toán tử so sánh khác nhau. Đó là lý do ta cần quét `!=` như một lexeme duy nhất. Tương tự, `<`, `>`, và `=` đều có thể theo sau bởi `=` để tạo ra các toán tử so sánh và bằng khác.

Với tất cả các trường hợp này, ta cần nhìn vào ký tự thứ hai.

^code two-char-tokens (1 before, 2 after)

Các trường hợp đó dùng phương thức mới này:

^code match

Nó giống như một `advance()` có điều kiện. Ta chỉ tiêu thụ ký tự hiện tại nếu nó đúng là ký tự ta đang tìm.

Dùng `match()`, ta nhận diện các lexeme này theo hai bước. Khi gặp, ví dụ, `!`, ta nhảy vào case tương ứng trong switch. Điều đó nghĩa là ta biết lexeme *bắt đầu* bằng `!`. Sau đó, ta nhìn ký tự tiếp theo để xác định xem đó là `!=` hay chỉ là `!`.

## Lexeme dài hơn

Ta vẫn còn thiếu một toán tử: `/` cho phép chia. Ký tự này cần xử lý đặc biệt một chút vì comment cũng bắt đầu bằng dấu gạch chéo.

^code slash (1 before, 2 after)

Điều này tương tự như các toán tử hai ký tự khác, ngoại trừ khi ta tìm thấy dấu `/` thứ hai, ta chưa kết thúc token ngay. Thay vào đó, ta tiếp tục tiêu thụ ký tự cho đến khi gặp cuối dòng.

Đây là chiến lược chung của ta để xử lý các lexeme dài hơn. Sau khi phát hiện phần bắt đầu của một lexeme, ta chuyển sang đoạn code chuyên biệt cho lexeme đó để tiếp tục “ăn” ký tự cho đến khi gặp điểm kết thúc.

Ta có thêm một helper khác:

^code peek

Nó giống như `advance()`, nhưng không tiêu thụ ký tự. Điều này được gọi là <span name="match">**lookahead**</span>. Vì nó chỉ nhìn vào ký tự hiện tại chưa tiêu thụ, nên ta có *lookahead một ký tự*. Số này càng nhỏ thì scanner thường chạy càng nhanh. Các quy tắc của lexical grammar quyết định ta cần lookahead bao nhiêu. May mắn thay, hầu hết các ngôn ngữ phổ biến chỉ cần nhìn trước một hoặc hai ký tự.

<aside name="match">

Về mặt kỹ thuật, `match()` cũng đang thực hiện lookahead. `advance()` và `peek()` là các toán tử cơ bản, còn `match()` kết hợp chúng lại.

</aside>

Comment là lexeme, nhưng chúng không mang ý nghĩa, và parser không muốn xử lý chúng. Vì vậy, khi đến cuối comment, ta *không* gọi `addToken()`. Khi vòng lặp quay lại để bắt đầu lexeme tiếp theo, `start` được đặt lại và lexeme của comment biến mất như một làn khói.

Nhân tiện, đây cũng là lúc tốt để bỏ qua những ký tự vô nghĩa khác: xuống dòng và khoảng trắng.

^code whitespace (1 before, 3 after)

Khi gặp khoảng trắng, ta đơn giản quay lại đầu vòng lặp quét. Điều đó bắt đầu một lexeme mới *sau* ký tự khoảng trắng. Với xuống dòng, ta làm tương tự, nhưng cũng tăng bộ đếm dòng. (Đây là lý do ta dùng `peek()` để tìm ký tự xuống dòng kết thúc comment thay vì `match()`. Ta muốn ký tự xuống dòng đó đưa ta đến đây để cập nhật `line`.)

Scanner của ta đang trở nên thông minh hơn. Nó có thể xử lý code khá tự do như:

```lox
// this is a comment
(( )){} // grouping stuff
!*+-/=<> <= == // operators
```

### String literal

Giờ ta đã quen với lexeme dài hơn, ta sẵn sàng xử lý literal. Ta sẽ làm chuỗi trước, vì chúng luôn bắt đầu bằng một ký tự cụ thể, `"`.

^code string-start (1 before, 2 after)

Hàm này gọi tới:

^code string

Giống như với comment, ta tiêu thụ ký tự cho đến khi gặp dấu `"` kết thúc chuỗi. Ta cũng xử lý gọn gàng trường hợp hết input trước khi chuỗi được đóng và báo lỗi cho tình huống đó.

Không vì lý do đặc biệt nào, Lox hỗ trợ chuỗi nhiều dòng. Điều này có ưu và nhược điểm, nhưng việc cấm chúng phức tạp hơn một chút so với cho phép, nên tôi để nguyên. Điều đó có nghĩa là ta cũng cần cập nhật `line` khi gặp xuống dòng bên trong chuỗi.

Cuối cùng, điểm thú vị là khi tạo token, ta cũng tạo ra *giá trị* chuỗi thực tế sẽ được interpreter sử dụng sau này. Ở đây, việc chuyển đổi chỉ cần một `substring()` để bỏ dấu ngoặc kép bao quanh. Nếu Lox hỗ trợ escape sequence như `\n`, ta sẽ giải mã chúng ở đây.

### Number literal

Tất cả số trong Lox đều là số thực dấu chấm động ở runtime, nhưng cả literal nguyên và thập phân đều được hỗ trợ. Một number literal là một chuỗi <span name="minus">chữ số</span> tùy chọn theo sau bởi một dấu `.` và một hoặc nhiều chữ số tiếp theo.

<aside name="minus">

Vì ta chỉ tìm chữ số để bắt đầu một số, điều đó có nghĩa là `-123` không phải là number *literal*. Thay vào đó, `-123` là một *biểu thức* áp dụng toán tử `-` lên number literal `123`. Trên thực tế, kết quả là như nhau, dù có một trường hợp đặc biệt thú vị nếu ta thêm lời gọi method trên số. Xem ví dụ:

```lox
print -123.abs();
```

Lệnh này in ra `-123` vì phép phủ định có độ ưu tiên thấp hơn lời gọi method. Ta có thể sửa bằng cách làm cho `-` trở thành một phần của number literal. Nhưng hãy xem xét:

```lox
var n = 123;
print -n.abs();
```

Lệnh này vẫn in ra `-123`, nên giờ ngôn ngữ trông không nhất quán. Dù làm gì, cũng sẽ có trường hợp trở nên kỳ lạ.

</aside>

```lox
1234
12.34
```

Ta không cho phép dấu chấm ở đầu hoặc cuối, nên cả hai trường hợp này đều không hợp lệ:

```lox
.1234
1234.
```

Ta có thể dễ dàng hỗ trợ trường hợp đầu, nhưng tôi bỏ qua để giữ mọi thứ đơn giản. Trường hợp thứ hai trở nên rắc rối nếu ta muốn cho phép method trên số như `123.sqrt()`.

Để nhận diện phần bắt đầu của một number lexeme, ta tìm bất kỳ chữ số nào. Việc thêm case cho từng chữ số thập phân khá tẻ nhạt, nên ta sẽ nhét nó vào case mặc định.

^code digit-start (1 before, 1 after)

Điều này dựa vào tiện ích nhỏ này:

^code is-digit

<aside name="is-digit">

Thư viện chuẩn Java cung cấp [`Character.isDigit()`](http://docs.oracle.com/javase/7/docs/api/java/lang/Character.html#isDigit(char)), có vẻ phù hợp. Tiếc là phương thức này cho phép những thứ như chữ số Devanagari, số full-width, và các ký tự lạ khác mà ta không muốn.

</aside>

Khi đã biết mình đang ở trong một số, ta sẽ rẽ sang một phương thức riêng để tiêu thụ phần còn lại của literal, giống như cách ta làm với chuỗi.

^code number

Ta tiêu thụ càng nhiều chữ số càng tốt cho phần nguyên của literal. Sau đó, ta tìm phần thập phân, tức là một dấu chấm (`.`) theo sau bởi ít nhất một chữ số. Nếu có phần thập phân, ta lại tiếp tục tiêu thụ càng nhiều chữ số càng tốt.

Nhìn vượt qua dấu chấm đòi hỏi lookahead hai ký tự, vì ta không muốn tiêu thụ dấu `.` cho đến khi chắc chắn có một chữ số *sau* nó. Vậy nên ta thêm:

^code peek-next

<aside name="peek-next">

Tôi có thể đã làm cho `peek()` nhận một tham số chỉ số ký tự cần nhìn trước thay vì định nghĩa hai hàm riêng, nhưng như vậy sẽ cho phép lookahead *tùy ý xa*. Việc cung cấp hai hàm riêng giúp người đọc code hiểu rõ rằng scanner của ta chỉ nhìn trước tối đa hai ký tự.

</aside>

Cuối cùng, ta chuyển đổi lexeme thành giá trị số. Interpreter của ta dùng kiểu `Double` của Java để biểu diễn số, nên ta tạo ra một giá trị thuộc kiểu đó. Ta dùng chính phương thức parse của Java để chuyển lexeme thành một số thực Java. Ta có thể tự hiện thực việc này, nhưng thật lòng mà nói, trừ khi bạn đang ôn gấp cho một buổi phỏng vấn lập trình sắp tới, thì không đáng tốn thời gian.

Các literal còn lại là Boolean và `nil`, nhưng ta xử lý chúng như keyword, và điều đó dẫn ta tới...

## Reserved Word & Identifier

Scanner của ta gần như xong. Phần còn lại của lexical grammar cần hiện thực là identifier và “họ hàng” gần của chúng — các reserved word. Bạn có thể nghĩ rằng ta có thể khớp keyword như `or` theo cùng cách ta xử lý các toán tử nhiều ký tự như `<=`.

```java
case 'o':
  if (match('r')) {
    addToken(OR);
  }
  break;
```

Hãy nghĩ xem điều gì sẽ xảy ra nếu người dùng đặt tên biến là `orchid`. Scanner sẽ thấy hai chữ cái đầu `or` và lập tức phát ra token keyword `or`. Điều này dẫn ta tới một nguyên tắc quan trọng gọi là <span name="maximal">**maximal munch**</span>. Khi hai quy tắc của lexical grammar đều có thể khớp một đoạn code mà scanner đang xét, *quy tắc nào khớp được nhiều ký tự hơn sẽ thắng*.

Nguyên tắc này nói rằng nếu ta có thể khớp `orchid` như một identifier và `or` như một keyword, thì trường hợp đầu sẽ thắng. Đây cũng là lý do ta ngầm giả định trước đó rằng `<=` nên được quét thành một token `<=` duy nhất chứ không phải `<` rồi `=`.

<aside name="maximal">

Hãy xem đoạn code C “khó chịu” này:

```c
---a;
```

Nó có hợp lệ không? Điều đó phụ thuộc vào cách scanner tách lexeme. Nếu scanner thấy nó như thế này:

```c
- --a;
```

Thì nó có thể được parse. Nhưng điều đó sẽ yêu cầu scanner phải biết về cấu trúc ngữ pháp của code xung quanh, khiến mọi thứ rối rắm hơn ta muốn. Thay vào đó, nguyên tắc maximal munch nói rằng nó sẽ *luôn* được quét như:

```c
-- -a;
```

Nó quét như vậy ngay cả khi điều đó dẫn tới lỗi cú pháp sau này trong parser.

</aside>

Maximal munch có nghĩa là ta không thể dễ dàng phát hiện một reserved word cho đến khi đã tới cuối của thứ có thể là một identifier. Suy cho cùng, một reserved word *là* một identifier, chỉ là một identifier đã bị ngôn ngữ “giữ chỗ” để dùng cho mục đích riêng. Đó là lý do có thuật ngữ **reserved word**.

Vậy nên ta bắt đầu bằng cách giả định bất kỳ lexeme nào bắt đầu bằng chữ cái hoặc dấu gạch dưới là một identifier.

^code identifier-start (3 before, 3 after)

Phần còn lại của code nằm ở đây:

^code identifier

Ta định nghĩa nó dựa trên các helper sau:

^code is-alpha

Như vậy là identifier đã hoạt động. Để xử lý keyword, ta kiểm tra xem lexeme của identifier có nằm trong danh sách reserved word hay không. Nếu có, ta dùng loại token đặc biệt cho keyword đó. Ta định nghĩa tập hợp reserved word trong một map.

^code keyword-map

Sau đó, sau khi quét một identifier, ta kiểm tra xem nó có khớp với mục nào trong map không.

^code keyword-type (2 before, 1 after)

Nếu có, ta dùng loại token của keyword đó. Nếu không, nó là một identifier do người dùng định nghĩa.

Và với điều đó, ta đã có một scanner hoàn chỉnh cho toàn bộ lexical grammar của Lox. Hãy mở REPL và gõ vào một số code hợp lệ và không hợp lệ. Nó có tạo ra các token như bạn mong đợi không? Hãy thử nghĩ ra vài trường hợp đặc biệt thú vị và xem nó xử lý chúng có đúng như mong muốn không.

<div class="challenges">

## Thử thách

1.  Lexical grammar của Python và Haskell không phải là *regular*. Điều đó có nghĩa là gì, và tại sao lại như vậy?

1.  Ngoài việc tách token — phân biệt `print foo` với `printfoo` — khoảng trắng không được dùng nhiều trong hầu hết các ngôn ngữ. Tuy nhiên, ở một vài “góc tối”, khoảng trắng *có* ảnh hưởng đến cách code được parse trong CoffeeScript, Ruby, và C preprocessor. Ở mỗi ngôn ngữ đó, nó xuất hiện ở đâu và có tác động gì?

1.  Scanner của ta ở đây, giống như hầu hết các scanner khác, loại bỏ comment và khoảng trắng vì parser không cần chúng. Tại sao bạn có thể muốn viết một scanner *không* loại bỏ những thứ đó? Nó sẽ hữu ích cho mục đích gì?

1.  Thêm hỗ trợ cho scanner của Lox để xử lý comment khối kiểu C `/* ... */`. Đảm bảo xử lý cả xuống dòng bên trong chúng. Cân nhắc cho phép chúng lồng nhau. Việc thêm hỗ trợ lồng nhau có tốn nhiều công sức hơn bạn nghĩ không? Tại sao?

</div>

<div class="design-note">

## Ghi chú thiết kế: Dấu chấm phẩy ngầm định

Lập trình viên ngày nay có rất nhiều lựa chọn ngôn ngữ và trở nên khắt khe hơn về cú pháp. Họ muốn ngôn ngữ của mình trông gọn gàng và hiện đại. Một “mảng rêu” cú pháp mà hầu như mọi ngôn ngữ mới đều cạo bỏ (và một số ngôn ngữ cổ như BASIC chưa bao giờ có) là `;` như một ký hiệu kết thúc câu lệnh tường minh.

Thay vào đó, họ coi xuống dòng như một ký hiệu kết thúc câu lệnh ở những nơi hợp lý. Phần “ở những nơi hợp lý” mới là phần khó. Dù *hầu hết* câu lệnh nằm trên một dòng riêng, đôi khi bạn cần trải một câu lệnh ra vài dòng. Những dấu xuống dòng xen giữa đó không nên bị coi là kết thúc câu lệnh.

Hầu hết các trường hợp rõ ràng mà xuống dòng nên bị bỏ qua thì dễ phát hiện, nhưng vẫn có một vài tình huống “khó chịu”:

* Giá trị trả về nằm ở dòng tiếp theo:

    ```js
    if (condition) return
    "value"
    ```

    `"value"` là giá trị được trả về, hay ta có một câu lệnh `return` không giá trị theo sau bởi một câu lệnh biểu thức chứa string literal?

* Biểu thức trong ngoặc ở dòng tiếp theo:

    ```js
    func
    (parenthesized)
    ```

    Đây là lời gọi `func(parenthesized)`, hay là hai câu lệnh biểu thức, một cho `func` và một cho biểu thức trong ngoặc?

* Dấu `-` ở dòng tiếp theo:

    ```js
    first
    -second
    ```

    Đây là `first - second` — phép trừ infix — hay là hai câu lệnh biểu thức, một cho `first` và một để phủ định `second`?

Trong tất cả các trường hợp này, dù coi xuống dòng là dấu phân cách hay không đều tạo ra code hợp lệ, nhưng có thể không phải là code người dùng mong muốn. Giữa các ngôn ngữ, có một sự đa dạng đáng lo ngại về các quy tắc dùng để quyết định xuống dòng nào là dấu phân cách. Dưới đây là một vài ví dụ:

*   [Lua][] hoàn toàn bỏ qua xuống dòng, nhưng kiểm soát cú pháp cẩn thận để hầu như không cần dấu phân cách giữa các câu lệnh. Điều này hoàn toàn hợp lệ:

    ```lua
    a = 1 b = 2
    ```

    Lua tránh vấn đề `return` bằng cách yêu cầu câu lệnh `return` phải là câu lệnh cuối cùng trong một block. Nếu có giá trị sau `return` trước từ khóa `end`, nó *phải* là giá trị trả về. Với hai trường hợp còn lại, họ cho phép dùng `;` tường minh và mong người dùng sẽ dùng nó. Trên thực tế, điều đó hầu như không xảy ra vì chẳng mấy khi có câu lệnh biểu thức dạng ngoặc hoặc phủ định đơn ngôi.

*   [Go][] xử lý xuống dòng ngay trong scanner. Nếu một xuống dòng xuất hiện sau một số loại token được biết là có thể kết thúc câu lệnh, xuống dòng đó được coi như dấu chấm phẩy. Ngược lại, nó bị bỏ qua. Nhóm phát triển Go cung cấp một trình định dạng code chuẩn, [gofmt][], và cộng đồng rất nhiệt tình sử dụng nó, đảm bảo rằng code theo phong cách idiomatic hoạt động tốt với quy tắc đơn giản này.

*   [Python][] coi mọi xuống dòng là có ý nghĩa trừ khi có dấu gạch chéo ngược ở cuối dòng để nối sang dòng tiếp theo. Tuy nhiên, xuống dòng bên trong cặp ngoặc (`()`, `[]`, hoặc `{}`) sẽ bị bỏ qua. Phong cách idiomatic rất ưa chuộng cách thứ hai.

    Quy tắc này hoạt động tốt với Python vì đây là ngôn ngữ định hướng câu lệnh mạnh. Đặc biệt, cú pháp của Python đảm bảo một câu lệnh không bao giờ xuất hiện bên trong một biểu thức. C cũng làm vậy, nhưng nhiều ngôn ngữ khác có cú pháp “lambda” hoặc function literal thì không.

    Ví dụ trong JavaScript:

    ```js
    console.log(function() {
      statement();
    });
    ```

    Ở đây, biểu thức `console.log()` chứa một function literal, và bên trong nó lại chứa *câu lệnh* `statement();`.

    Python sẽ cần một bộ quy tắc khác để nối dòng ngầm định nếu bạn có thể quay lại *bên trong* một <span name="lambda">câu lệnh</span> nơi xuống dòng trở nên có ý nghĩa trong khi vẫn đang lồng bên trong ngoặc.

<aside name="lambda">

Và giờ bạn đã biết tại sao `lambda` của Python chỉ cho phép một biểu thức duy nhất trong thân.

</aside>

*   Quy tắc “[automatic semicolon insertion][asi]” của JavaScript mới thực sự kỳ lạ. Trong khi các ngôn ngữ khác giả định hầu hết xuống dòng *có* ý nghĩa và chỉ một số ít bị bỏ qua trong câu lệnh nhiều dòng, JS lại giả định ngược lại. Nó coi tất cả xuống dòng của bạn là khoảng trắng vô nghĩa *trừ khi* gặp lỗi parse. Nếu gặp lỗi, nó quay lại và thử biến xuống dòng trước đó thành dấu chấm phẩy để có code hợp lệ về mặt cú pháp.

    Ghi chú thiết kế này sẽ biến thành một “bài luận” nếu tôi đi sâu vào chi tiết cách nó *hoạt động*, chứ chưa nói đến tất cả những lý do khiến “giải pháp” của JavaScript là một ý tưởng tồi. Nó là một mớ hỗn độn. JavaScript là ngôn ngữ duy nhất tôi biết mà nhiều hướng dẫn phong cách yêu cầu dấu chấm phẩy tường minh sau mỗi câu lệnh, dù về lý thuyết ngôn ngữ cho phép bạn bỏ chúng.

Nếu bạn đang thiết kế một ngôn ngữ mới, gần như chắc chắn bạn *nên* tránh ký hiệu kết thúc câu lệnh tường minh. Lập trình viên cũng là những sinh vật chạy theo xu hướng như bao người khác, và dấu chấm phẩy đã lỗi thời như CÁC TỪ KHÓA VIẾT HOA TOÀN BỘ. Chỉ cần đảm bảo bạn chọn một bộ quy tắc hợp lý cho cú pháp và phong cách của ngôn ngữ mình. Và đừng làm như JavaScript đã làm.

</div>

[lua]: https://www.lua.org/pil/1.1.html  
[go]: https://golang.org/ref/spec#Semicolons  
[gofmt]: https://golang.org/cmd/gofmt/  
[python]: https://docs.python.org/3.5/reference/lexical_analysis.html#implicit-line-joining  
[asi]: https://www.ecma-international.org/ecma-262/5.1/#sec-7.9  