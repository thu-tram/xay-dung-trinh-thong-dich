> Giữa chặng đường đời của chúng ta, tôi thấy mình lạc giữa khu rừng tăm tối  
> nơi con đường thẳng đã mất dấu.  
>
> <cite>Dante Alighieri, <em>Inferno</em></cite>

Chương này thật sự thú vị, không chỉ vì một, hay hai, mà là *ba* lý do.  
Thứ nhất, nó mang đến mảnh ghép cuối cùng trong pipeline execute của VM. Khi đã hoàn thiện, chúng ta có thể dẫn luồng source code của người dùng từ bước scanning cho đến khi execute.

<img src="image/compiling-expressions/pipeline.png" alt="Hạ phần 'compiler' của đường ống giữa 'scanner' và 'VM'." />

Thứ hai, chúng ta sẽ viết một *compiler* thực thụ. Nó sẽ parse source code và xuất ra một chuỗi các lệnh nhị phân cấp thấp. Ừ thì, đó là <span name="wirth">bytecode</span> chứ không phải tập lệnh gốc của một con chip nào, nhưng nó gần “sát kim loại” hơn nhiều so với jlox. Chúng ta sắp trở thành những kẻ “hack” ngôn ngữ thực sự.

<aside name="wirth">

Bytecode đã đủ tốt cho Niklaus Wirth, và chẳng ai nghi ngờ uy tín của ông cả.

</aside>

<span name="pratt">Thứ ba</span> và cuối cùng, tôi sẽ giới thiệu với bạn một trong những thuật toán mà tôi yêu thích nhất: “top-down operator precedence parsing” của Vaughan Pratt. Đây là cách thanh nhã nhất mà tôi biết để parse biểu thức. Nó xử lý gọn gàng các toán tử prefix, postfix, infix, *mixfix*, bất kỳ dạng *-fix* nào bạn có. Nó giải quyết precedence và associativity một cách nhẹ nhàng. Tôi thật sự rất thích nó.

<aside name="pratt">

Pratt parser giống như một truyền thống truyền miệng trong ngành. Chưa từng có cuốn sách về compiler hay ngôn ngữ nào mà tôi đọc dạy về nó. Giới học thuật thì tập trung vào parser sinh tự động, còn kỹ thuật của Pratt dành cho parser viết tay, nên thường bị bỏ qua.

Nhưng trong các compiler dùng thực tế, nơi parser viết tay khá phổ biến, bạn sẽ ngạc nhiên khi thấy nhiều người biết đến nó. Hỏi họ học từ đâu, câu trả lời thường là: “Ồ, tôi từng làm cho một compiler nhiều năm trước, và đồng nghiệp bảo họ lấy nó từ một front end cũ nào đó…”

</aside>

Như thường lệ, trước khi đến phần thú vị, chúng ta cần xử lý vài bước chuẩn bị. Bạn phải ăn rau trước khi được ăn tráng miệng. Đầu tiên, hãy bỏ đi phần scaffolding tạm thời mà ta viết để test scanner và thay bằng thứ gì đó hữu ích hơn.

^code interpret-chunk (1 before, 1 after)

Chúng ta tạo một chunk rỗng mới và chuyển nó cho compiler. Compiler sẽ lấy chương trình của người dùng và lấp đầy chunk bằng bytecode. Ít nhất, đó là những gì nó sẽ làm nếu chương trình không có lỗi compile. Nếu gặp lỗi, `compile()` sẽ trả về `false` và chúng ta bỏ chunk đó đi vì không dùng được.

Ngược lại, nếu compile thành công, chúng ta gửi chunk hoàn chỉnh sang VM để execute. Khi VM chạy xong, chúng ta giải phóng chunk và kết thúc. Như bạn thấy, signature của `compile()` giờ đã khác.

^code compile-h (2 before, 2 after)

Chúng ta truyền vào chunk nơi compiler sẽ ghi code, và `compile()` trả về kết quả compile thành công hay không. Chúng ta cũng thay đổi signature tương tự trong phần implementation.

^code compile-signature (2 before, 1 after)

Lệnh gọi `initScanner()` là dòng duy nhất còn lại từ chương này. Hãy gỡ bỏ code tạm mà ta viết để test scanner và thay bằng ba dòng này:

^code compile-chunk (1 before, 1 after)

Lệnh gọi `advance()` sẽ “mồi bơm” cho scanner. Chúng ta sẽ sớm thấy nó làm gì. Sau đó, ta parse một biểu thức duy nhất. Chúng ta chưa xử lý statement, nên đây là phần duy nhất của grammar được hỗ trợ. Chúng ta sẽ quay lại khi [thêm statement trong vài chương tới](global-variables.html). Sau khi compile xong biểu thức, ta phải ở cuối source code, nên ta kiểm tra token EOF.

Chúng ta sẽ dành phần còn lại của chương này để làm cho hàm này hoạt động, đặc biệt là lời gọi `expression()` kia. Thông thường, ta sẽ nhảy ngay vào định nghĩa hàm đó và triển khai từ trên xuống dưới.

Nhưng chương này thì <span name="blog">khác</span>. Kỹ thuật parsing của Pratt rất đơn giản khi bạn đã nắm toàn bộ trong đầu, nhưng hơi khó để chia thành từng phần nhỏ. Nó đệ quy, tất nhiên, đó là một phần của vấn đề. Nhưng nó cũng dựa vào một bảng dữ liệu lớn. Khi ta xây dựng thuật toán, bảng này sẽ có thêm nhiều cột.

<aside name="blog">

Nếu chương này chưa khiến bạn “ngấm” và bạn muốn một góc nhìn khác, tôi đã viết một bài hướng dẫn cùng thuật toán này nhưng dùng Java và phong cách lập trình hướng đối tượng: ["Pratt Parsing: Expression Parsing Made Easy"](http://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/).

</aside>

Tôi không muốn phải xem lại hơn 40 dòng code mỗi khi mở rộng bảng. Thế nên, chúng ta sẽ tiếp cận lõi của parser từ bên ngoài, xử lý tất cả phần xung quanh trước khi vào phần “ruột” hấp dẫn. Điều này sẽ đòi hỏi bạn kiên nhẫn hơn và dành chút “bộ nhớ tạm” trong đầu, nhưng đây là cách tốt nhất mà tôi có thể nghĩ ra.

## Biên dịch một lượt (Single-Pass Compilation)

Một compiler thường có hai nhiệm vụ chính. Nó parse source code của người dùng để hiểu ý nghĩa của nó. Sau đó, nó dùng kiến thức đó để xuất ra các lệnh cấp thấp tạo ra cùng một ngữ nghĩa. Nhiều ngôn ngữ tách hai vai trò này thành hai <span name="passes">pass</span> riêng biệt trong quá trình triển khai. Một parser tạo ra AST — giống như jlox đã làm — rồi một code generator sẽ duyệt AST và xuất ra target code.

<aside name="passes">

Thực tế, hầu hết các compiler tối ưu hóa phức tạp còn có nhiều hơn rất nhiều so với chỉ hai pass. Việc xác định không chỉ *có* những pass tối ưu nào, mà còn sắp xếp thứ tự chúng ra sao để vắt kiệt hiệu năng của compiler — vì các tối ưu hóa thường tương tác với nhau theo những cách phức tạp — nằm đâu đó giữa “một lĩnh vực nghiên cứu mở” và “một môn hắc ám”.

</aside>

Trong clox, chúng ta sẽ áp dụng cách tiếp cận kiểu “old-school” và gộp hai pass này thành một. Ngày xưa, các “hacker” ngôn ngữ làm vậy vì máy tính khi đó thực sự không đủ bộ nhớ để lưu toàn bộ AST của một file source. Còn chúng ta làm vậy vì nó giúp compiler đơn giản hơn, điều này rất có lợi khi lập trình bằng C.

Những compiler single-pass như chúng ta sắp xây không phù hợp với mọi ngôn ngữ. Vì compiler chỉ có “tầm nhìn qua lỗ khóa” vào chương trình của người dùng khi sinh code, ngôn ngữ phải được thiết kế sao cho bạn không cần quá nhiều ngữ cảnh xung quanh để hiểu một đoạn cú pháp. May mắn thay, Lox nhỏ gọn, kiểu động lại <span name="lox">rất phù hợp</span> với điều đó.

<aside name="lox">

Điều này chắc cũng không làm bạn ngạc nhiên. Dù sao thì tôi cũng thiết kế ngôn ngữ này đặc biệt cho cuốn sách này mà.

<img src="image/compiling-expressions/keyhole.png" alt="Nhìn qua lỗ khóa thấy 'var x;'" />

</aside>

Điều này có nghĩa là, về mặt thực tế, module C “compiler” của chúng ta sẽ có những chức năng bạn sẽ nhận ra từ jlox để parsing — tiêu thụ token, khớp loại token mong đợi, v.v. — và cũng có các hàm cho code gen — phát bytecode và thêm constant vào chunk đích. (Và điều đó cũng có nghĩa là tôi sẽ dùng “parsing” và “compiling” thay thế cho nhau xuyên suốt chương này và các chương sau.)

Chúng ta sẽ xây dựng phần parsing và phần code generation trước. Sau đó, chúng ta sẽ nối chúng lại với đoạn code ở giữa, đoạn này dùng kỹ thuật của Pratt để parse grammar đặc trưng của Lox và xuất ra bytecode đúng.

## Parsing Token

Bắt đầu với nửa đầu của compiler. Tên hàm này chắc nghe quen lắm.

^code advance (1 before)

Giống như trong jlox, nó bước tới token tiếp theo trong luồng token. Nó yêu cầu scanner lấy token kế tiếp và lưu lại để dùng sau. Trước khi làm vậy, nó lấy token `current` cũ và cất vào trường `previous`. Điều này sẽ hữu ích sau này khi chúng ta cần lấy lexeme sau khi đã match một token.

Code để đọc token tiếp theo được bao trong một vòng lặp. Nhớ rằng, scanner của clox không báo lỗi lexical. Thay vào đó, nó tạo ra các *error token* đặc biệt và để parser chịu trách nhiệm báo lỗi. Chúng ta xử lý điều đó ở đây.

Chúng ta tiếp tục lặp, đọc token và báo lỗi, cho đến khi gặp một token không lỗi hoặc tới cuối file. Bằng cách này, phần còn lại của parser chỉ thấy token hợp lệ. Token hiện tại và token trước đó được lưu trong struct này:

^code parser (1 before, 2 after)

Giống như ở các module khác, chúng ta có một biến global duy nhất của kiểu struct này để không phải truyền trạng thái qua lại giữa các hàm trong compiler.

### Xử lý lỗi cú pháp

Nếu scanner đưa cho chúng ta một error token, chúng ta cần báo cho người dùng biết. Điều đó được thực hiện bằng:

^code error-at-current

Chúng ta lấy vị trí từ token hiện tại để báo cho người dùng lỗi xảy ra ở đâu và chuyển tiếp nó tới `errorAt()`. Thường thì, chúng ta sẽ báo lỗi tại vị trí của token vừa tiêu thụ, nên chúng ta đặt tên ngắn hơn cho hàm khác này:

^code error

Phần xử lý thực sự diễn ra ở đây:

^code error-at

Đầu tiên, chúng ta in ra vị trí lỗi. Chúng ta cố gắng hiển thị lexeme nếu nó dễ đọc với con người. Sau đó, chúng ta in thông báo lỗi. Tiếp theo, chúng ta đặt flag `hadError`. Cờ này ghi nhận việc có lỗi xảy ra trong quá trình compile hay không. Trường này cũng nằm trong struct parser.

^code had-error-field (1 before, 1 after)

Lúc trước tôi đã nói rằng `compile()` nên trả về `false` nếu có lỗi. Giờ chúng ta có thể làm điều đó.

^code return-had-error (1 before, 1 after)

Tôi còn một flag nữa để giới thiệu cho việc xử lý lỗi. Chúng ta muốn tránh “lỗi dây chuyền”. Nếu người dùng mắc lỗi trong code và parser bị lạc vị trí trong grammar, chúng ta không muốn nó tuôn ra một đống lỗi vô nghĩa sau lỗi đầu tiên.

Chúng ta đã xử lý điều này trong jlox bằng panic mode error recovery. Trong Java interpreter, chúng ta ném một exception để thoát ra khỏi toàn bộ code parser tới một điểm mà ta có thể bỏ qua token và đồng bộ lại. Trong C, chúng ta không có <span name="setjmp">exception</span>. Thay vào đó, chúng ta sẽ dùng một chút “ảo thuật”. Chúng ta thêm một flag để theo dõi xem hiện tại có đang ở panic mode hay không.

<aside name="setjmp">

Cũng có `setjmp()` và `longjmp()`, nhưng tôi không muốn đụng vào. Chúng khiến việc rò rỉ bộ nhớ, quên duy trì các bất biến, hoặc gặp một Ngày Tồi Tệ trở nên quá dễ dàng.

</aside>

^code panic-mode-field (1 before, 1 after)

Khi xảy ra lỗi, chúng ta bật flag này.

^code set-panic-mode (1 before, 1 after)

Sau đó, chúng ta cứ tiếp tục compile như bình thường, như thể lỗi chưa từng xảy ra. Bytecode sẽ không bao giờ được execute, nên việc tiếp tục cũng không gây hại gì. Mẹo ở đây là khi flag panic mode đang bật, chúng ta đơn giản bỏ qua mọi lỗi khác được phát hiện.

^code check-panic-mode (1 before, 1 after)

Có khả năng parser sẽ “đi lạc vào bụi rậm”, nhưng người dùng sẽ không biết vì tất cả lỗi đều bị nuốt mất. Panic mode kết thúc khi parser tới một điểm đồng bộ. Với Lox, chúng ta chọn ranh giới statement, nên khi sau này thêm statement vào compiler, chúng ta sẽ tắt flag ở đó.

Những trường mới này cần được khởi tạo.

^code init-parser-error (1 before, 1 after)

Và để hiển thị lỗi, chúng ta cần một header chuẩn.

^code compiler-include-stdlib (1 before, 2 after)

Còn một hàm parsing cuối cùng, một “người bạn cũ” từ jlox.

^code consume

Nó giống `advance()` ở chỗ đọc token tiếp theo. Nhưng nó cũng kiểm tra token đó có đúng loại mong đợi hay không. Nếu không, nó báo lỗi. Hàm này là nền tảng của hầu hết các lỗi cú pháp trong compiler.

OK, vậy là đủ cho phần front-end lúc này.


## Sinh Bytecode (Emitting Bytecode)

Sau khi chúng ta parse và hiểu một phần chương trình của người dùng, bước tiếp theo là dịch nó thành một chuỗi các lệnh bytecode. Và nó bắt đầu bằng bước đơn giản nhất có thể: thêm một byte vào chunk.

^code emit-byte

Thật khó tin là những thứ “vĩ đại” lại có thể đi qua một hàm đơn giản như vậy. Nó ghi byte được truyền vào — byte này có thể là một opcode hoặc một toán hạng (operand) của một lệnh. Nó cũng gửi kèm thông tin dòng của token trước đó để khi xảy ra lỗi runtime, chúng ta biết nó gắn với dòng nào.

Chunk mà chúng ta đang ghi được truyền vào `compile()`, nhưng nó cần “đi” đến được `emitByte()`. Để làm điều đó, chúng ta dựa vào hàm trung gian này:

^code compiling-chunk (1 before, 1 after)

Hiện tại, con trỏ chunk được lưu trong một biến cấp module, giống như cách chúng ta lưu các trạng thái global khác. Sau này, khi bắt đầu compile các hàm do người dùng định nghĩa, khái niệm “current chunk” sẽ phức tạp hơn. Để tránh phải quay lại và sửa nhiều code, tôi đóng gói logic đó vào hàm `currentChunk()`.

Chúng ta khởi tạo biến module mới này trước khi ghi bất kỳ bytecode nào:

^code init-compile-chunk (2 before, 2 after)

Rồi, ở cuối cùng, khi compile xong chunk, chúng ta kết thúc mọi thứ.

^code finish-compile (1 before, 1 after)

Hàm này gọi đến:

^code end-compiler

Trong chương này, VM của chúng ta chỉ xử lý biểu thức. Khi bạn chạy clox, nó sẽ parse, compile và execute một biểu thức duy nhất, rồi in kết quả. Để in giá trị đó, tạm thời chúng ta dùng lệnh `OP_RETURN`. Vì vậy, compiler sẽ thêm một lệnh như vậy vào cuối chunk.

^code emit-return

Nhân tiện đang ở phần back end, ta cũng có thể làm cho cuộc sống dễ dàng hơn một chút.

^code emit-bytes

Theo thời gian, chúng ta sẽ gặp đủ nhiều trường hợp cần ghi một opcode theo sau bởi một toán hạng một byte, nên đáng để định nghĩa một hàm tiện ích như thế này.

## Parsing các biểu thức Prefix

Chúng ta đã tập hợp xong các hàm tiện ích cho parsing và code generation. Mảnh ghép còn thiếu là đoạn code ở giữa để kết nối chúng lại.

<img src="image/compiling-expressions/mystery.png" alt="Parsing functions ở bên trái, bytecode emitting functions ở bên phải. Ở giữa là gì?" />

Bước duy nhất trong `compile()` mà chúng ta còn chưa triển khai là hàm này:

^code expression

Chúng ta chưa sẵn sàng để triển khai mọi loại biểu thức trong Lox. Thậm chí, chúng ta còn chưa có Boolean. Trong chương này, chúng ta chỉ quan tâm đến bốn loại:

* Số literal: `123`
* Dấu ngoặc đơn để nhóm: `(123)`
* Phủ định đơn ngôi (unary negation): `-123`
* “Bốn kỵ sĩ” của số học: `+`, `-`, `*`, `/`

Khi chúng ta lần lượt viết hàm compile cho từng loại biểu thức này, chúng ta cũng sẽ xây dựng dần các yêu cầu cho table-driven parser sẽ gọi chúng.

### Parser cho token

Trước mắt, hãy tập trung vào các biểu thức Lox mà mỗi cái chỉ gồm một token duy nhất. Trong chương này, đó chỉ là số literal, nhưng sau này sẽ có thêm. Đây là cách chúng ta compile chúng:

Chúng ta ánh xạ mỗi loại token sang một loại biểu thức khác nhau. Chúng ta định nghĩa một hàm cho mỗi loại biểu thức để xuất ra bytecode phù hợp. Sau đó, chúng ta tạo một mảng con trỏ hàm. Các chỉ số trong mảng tương ứng với giá trị enum `TokenType`, và hàm ở mỗi chỉ số là code để compile một biểu thức của loại token đó.

Để compile số literal, chúng ta lưu con trỏ đến hàm sau tại chỉ số `TOKEN_NUMBER` trong mảng.

^code number

Chúng ta giả định token của số literal đã được tiêu thụ và lưu trong `previous`. Chúng ta lấy lexeme đó và dùng thư viện chuẩn C để chuyển nó thành giá trị double. Sau đó, chúng ta sinh code để load giá trị đó bằng hàm này:

^code emit-constant

Đầu tiên, chúng ta thêm giá trị vào bảng constant, rồi phát ra lệnh `OP_CONSTANT` để đẩy nó lên stack khi runtime. Để chèn một mục vào bảng constant, chúng ta dùng:

^code make-constant

Phần lớn công việc diễn ra trong `addConstant()`, hàm mà chúng ta đã định nghĩa ở [chương trước](chunks-of-bytecode.html). Hàm đó thêm giá trị được truyền vào cuối bảng constant của chunk và trả về chỉ số của nó. Nhiệm vụ chính của hàm mới này là đảm bảo chúng ta không có quá nhiều constant. Vì lệnh `OP_CONSTANT` dùng một byte cho toán hạng chỉ số, chúng ta chỉ có thể lưu và load tối đa <span name="256">256</span> constant trong một chunk.

<aside name="256">

Đúng là giới hạn này khá thấp. Nếu đây là một ngôn ngữ đầy đủ, chúng ta sẽ muốn thêm một lệnh khác như `OP_CONSTANT_16` để lưu chỉ số dưới dạng toán hạng hai byte, cho phép xử lý nhiều constant hơn khi cần.

Code để hỗ trợ điều đó không quá thú vị, nên tôi đã bỏ qua trong clox, nhưng bạn sẽ muốn VM của mình có thể mở rộng cho các chương trình lớn hơn.

</aside>

Về cơ bản, chỉ cần vậy thôi. Miễn là có code phù hợp để tiêu thụ token `TOKEN_NUMBER`, tra hàm `number()` trong mảng con trỏ hàm, rồi gọi nó, chúng ta đã có thể compile số literal thành bytecode.



### Dấu ngoặc đơn để nhóm (Parentheses for grouping)

Mảng con trỏ hàm parsing “tưởng tượng” của chúng ta sẽ tuyệt vời nếu mọi biểu thức chỉ dài đúng một token. Tiếc là hầu hết lại dài hơn. Tuy nhiên, nhiều biểu thức *bắt đầu* bằng một token cụ thể. Chúng ta gọi chúng là biểu thức *prefix*. Ví dụ, khi đang parse một biểu thức và token hiện tại là `(`, ta biết chắc mình đang gặp một biểu thức nhóm trong ngoặc.

Hóa ra mảng con trỏ hàm của chúng ta cũng xử lý được những trường hợp này. Hàm parsing cho một loại biểu thức có thể tiêu thụ thêm bất kỳ token nào nó muốn, giống như trong một recursive descent parser thông thường. Đây là cách xử lý dấu ngoặc:

^code grouping

Một lần nữa, chúng ta giả định dấu `(` ban đầu đã được tiêu thụ. Chúng ta <span name="recursive">gọi đệ quy</span> lại vào `expression()` để compile biểu thức bên trong ngoặc, rồi parse dấu `)` đóng ở cuối.

<aside name="recursive">

Pratt parser không phải là một recursive *descent* parser, nhưng nó vẫn đệ quy. Điều này là hiển nhiên vì bản thân grammar cũng đệ quy.

</aside>

Về phía back end, một biểu thức nhóm thực sự chẳng có gì cả. Chức năng duy nhất của nó là cú pháp — cho phép bạn chèn một biểu thức có precedence thấp vào chỗ mà precedence cao hơn được mong đợi. Vì vậy, nó không có ngữ nghĩa runtime riêng và không phát ra bytecode nào. Lời gọi `expression()` bên trong sẽ lo việc sinh bytecode cho biểu thức nằm trong ngoặc.

### Phủ định đơn ngôi (Unary negation)

Dấu trừ đơn ngôi (unary minus) cũng là một biểu thức prefix, nên nó cũng hoạt động với mô hình của chúng ta.

^code unary

Token `-` ở đầu đã được tiêu thụ và đang nằm trong `parser.previous`. Chúng ta lấy loại token từ đó để biết mình đang xử lý toán tử unary nào. Hiện tại thì chưa cần thiết, nhưng điều này sẽ hợp lý hơn khi chúng ta dùng cùng hàm này để compile toán tử `!` trong [chương tiếp theo](types-of-values.html).

Giống như trong `grouping()`, chúng ta gọi đệ quy `expression()` để compile toán hạng. Sau đó, chúng ta phát bytecode để thực hiện phép phủ định. Có thể bạn thấy hơi lạ khi ghi lệnh negate *sau* bytecode của toán hạng, vì dấu `-` xuất hiện bên trái, nhưng hãy nghĩ theo thứ tự execute:

1. Ta đánh giá toán hạng trước, để lại giá trị của nó trên stack.  
2. Sau đó, ta pop giá trị đó, phủ định nó, rồi push kết quả.

Vì vậy, lệnh `OP_NEGATE` nên được phát ra <span name="line">cuối cùng</span>. Đây là một phần công việc của compiler — parse chương trình theo thứ tự xuất hiện trong source code và sắp xếp lại thành thứ tự execute.

<aside name="line">

Phát lệnh `OP_NEGATE` sau toán hạng đồng nghĩa với việc token hiện tại khi ghi bytecode *không* phải là token `-`. Điều này hầu như không quan trọng, ngoại trừ việc chúng ta dùng token đó để lấy số dòng gắn với lệnh.

Điều này có nghĩa là nếu bạn có một biểu thức phủ định nhiều dòng, như:

```lox
print -
  true;
```

Thì lỗi runtime sẽ được báo ở sai dòng. Ở đây, nó sẽ báo lỗi ở dòng 2, dù dấu `-` nằm ở dòng 1. Cách tiếp cận “chuẩn” hơn là lưu số dòng của token trước khi compile toán hạng, rồi truyền nó vào `emitByte()`, nhưng tôi muốn giữ mọi thứ đơn giản cho cuốn sách này.

</aside>

Tuy nhiên, có một vấn đề với đoạn code này. Hàm `expression()` mà nó gọi sẽ parse *bất kỳ* biểu thức nào làm toán hạng, bất kể precedence. Khi chúng ta thêm toán tử nhị phân và cú pháp khác, điều này sẽ gây lỗi. Xem ví dụ:

```lox
-a.b + c;
```

Ở đây, toán hạng của `-` chỉ nên là biểu thức `a.b`, không phải toàn bộ `a.b + c`. Nhưng nếu `unary()` gọi `expression()`, hàm này sẽ “ăn” hết phần còn lại, bao gồm cả `+`. Nó sẽ sai khi coi `-` có precedence thấp hơn `+`.

Khi parse toán hạng cho unary `-`, chúng ta cần compile chỉ những biểu thức ở một mức precedence nhất định hoặc cao hơn. Trong recursive descent parser của jlox, chúng ta làm điều đó bằng cách gọi vào hàm parse cho loại biểu thức có precedence thấp nhất mà ta muốn cho phép (trong trường hợp này là `call()`). Mỗi hàm parse một loại biểu thức cụ thể cũng sẽ parse luôn các loại có precedence cao hơn, nên bao trùm cả phần còn lại của bảng precedence.

Các hàm parsing như `number()` và `unary()` ở clox thì khác. Mỗi hàm chỉ parse đúng một loại biểu thức. Chúng không “xâu chuỗi” để bao gồm các loại có precedence cao hơn. Chúng ta cần một giải pháp khác, và nó trông như thế này:

^code parse-precedence

Hàm này — khi được triển khai — sẽ bắt đầu từ token hiện tại và parse bất kỳ biểu thức nào ở mức precedence cho trước hoặc cao hơn. Chúng ta sẽ cần chuẩn bị thêm trước khi viết thân hàm này, nhưng bạn có thể đoán rằng nó sẽ dùng bảng con trỏ hàm parsing mà tôi đã nhắc đến. Tạm thời, đừng lo quá nhiều về cách nó hoạt động. Để truyền “precedence” làm tham số, chúng ta định nghĩa nó dưới dạng số.

^code precedence (1 before, 2 after)

Đây là toàn bộ các mức precedence của Lox, từ thấp đến cao. Vì C tự động gán số tăng dần cho enum, điều này có nghĩa là `PREC_CALL` có giá trị số lớn hơn `PREC_UNARY`. Ví dụ, giả sử compiler đang xử lý đoạn code:

```lox
-a.b + c
```

Nếu ta gọi `parsePrecedence(PREC_ASSIGNMENT)`, nó sẽ parse toàn bộ biểu thức vì `+` có precedence cao hơn assignment. Nếu thay vào đó ta gọi `parsePrecedence(PREC_UNARY)`, nó sẽ compile `-a.b` và dừng lại. Nó không tiếp tục qua `+` vì phép cộng có precedence thấp hơn toán tử unary.

Với hàm này, việc bổ sung phần thân còn thiếu cho `expression()` trở nên dễ dàng.

^code expression-body (1 before, 1 after)

Chúng ta chỉ cần parse mức precedence thấp nhất, vốn bao gồm tất cả các biểu thức có precedence cao hơn. Giờ, để compile toán hạng cho một biểu thức unary, ta gọi hàm mới này và giới hạn ở mức phù hợp:

^code unary-operand (1 before, 2 after)

Chúng ta dùng chính precedence `PREC_UNARY` của toán tử unary để cho phép <span name="useful">lồng</span> các biểu thức unary như `!!doubleNegative`. Vì toán tử unary có precedence khá cao, điều này sẽ loại trừ đúng các toán tử nhị phân. Nói đến đây thì…

<aside name="useful">

Không phải việc lồng nhiều toán tử unary là đặc biệt hữu ích trong Lox. Nhưng các ngôn ngữ khác cho phép, nên chúng ta cũng cho phép.

</aside>

## Parsing các biểu thức Infix

Các toán tử nhị phân (binary operators) khác với những biểu thức trước đó vì chúng là *infix*. Với các biểu thức khác, chúng ta biết mình đang parse cái gì ngay từ token đầu tiên. Nhưng với biểu thức infix, ta sẽ không biết mình đang ở giữa một toán tử nhị phân cho đến *sau khi* đã parse toán hạng bên trái, rồi mới “đụng” vào token toán tử nằm ở giữa.

Ví dụ:

```lox
1 + 2
```

Hãy thử đi qua quá trình compile nó với những gì ta biết đến giờ:

1.  Ta gọi `expression()`. Hàm này lại gọi  
    `parsePrecedence(PREC_ASSIGNMENT)`.

2.  Hàm đó (khi được triển khai) sẽ thấy token số ở đầu và nhận ra ta đang parse một số literal. Nó chuyển quyền điều khiển sang `number()`.

3.  `number()` tạo một constant, phát lệnh `OP_CONSTANT`, rồi trả về `parsePrecedence()`.

Rồi sao nữa? Lời gọi `parsePrecedence()` phải tiêu thụ toàn bộ biểu thức cộng, nên nó cần tiếp tục bằng cách nào đó. May mắn là parser đang ở đúng vị trí ta cần. Sau khi compile xong biểu thức số ở đầu, token tiếp theo là `+`. Đây chính xác là token mà `parsePrecedence()` cần để phát hiện rằng ta đang ở giữa một biểu thức infix và nhận ra rằng biểu thức vừa compile thực ra là toán hạng của nó.

Vì vậy, mảng con trỏ hàm “tưởng tượng” này không chỉ liệt kê các hàm parse biểu thức bắt đầu bằng một token nhất định. Thay vào đó, nó là một *bảng* con trỏ hàm. Một cột ánh xạ các hàm parser prefix với loại token. Cột thứ hai ánh xạ các hàm parser infix với loại token.

Hàm mà chúng ta sẽ dùng làm infix parser cho `TOKEN_PLUS`, `TOKEN_MINUS`, `TOKEN_STAR` và `TOKEN_SLASH` là:

^code binary

Khi một hàm parser prefix được gọi, token dẫn đầu đã được tiêu thụ. Một hàm parser infix thì còn “*in medias res*” hơn nữa — toàn bộ biểu thức toán hạng bên trái đã được compile và toán tử infix tiếp theo cũng đã được tiêu thụ.

Việc toán hạng bên trái được compile trước là hoàn toàn ổn. Điều đó có nghĩa là khi runtime, đoạn code đó sẽ được execute trước. Khi chạy, giá trị nó tạo ra sẽ nằm trên stack — đúng chỗ mà toán tử infix cần.

Sau đó, ta đến `binary()` để xử lý phần còn lại của các toán tử số học. Hàm này compile toán hạng bên phải, tương tự như cách `unary()` compile toán hạng của nó. Cuối cùng, nó phát bytecode thực hiện phép toán nhị phân.

Khi chạy, VM sẽ execute code của toán hạng trái và phải, theo thứ tự đó, để lại giá trị của chúng trên stack. Sau đó, nó execute lệnh của toán tử. Lệnh này pop hai giá trị, tính toán, rồi push kết quả.

Đoạn code có lẽ khiến bạn chú ý ở đây là dòng `getRule()`. Khi parse toán hạng bên phải, ta lại phải quan tâm đến precedence. Xem ví dụ:

```lox
2 * 3 + 4
```

Khi parse toán hạng bên phải của biểu thức `*`, ta chỉ cần lấy `3`, chứ không phải `3 + 4`, vì `+` có precedence thấp hơn `*`. Ta có thể định nghĩa một hàm riêng cho mỗi toán tử nhị phân, mỗi hàm sẽ gọi `parsePrecedence()` và truyền vào mức precedence đúng cho toán hạng của nó.

Nhưng như vậy thì khá tẻ nhạt. Precedence của toán hạng bên phải của mỗi toán tử nhị phân luôn cao hơn <span name="higher">một mức</span> so với chính nó. Ta có thể tra điều đó một cách động bằng `getRule()` (sẽ nói sau). Dùng cách này, ta gọi `parsePrecedence()` với mức cao hơn một bậc so với toán tử hiện tại.

<aside name="higher">

Ta dùng mức precedence *cao hơn một bậc* cho toán hạng bên phải vì các toán tử nhị phân là left-associative. Với một chuỗi các toán tử *giống nhau*, như:

```lox
1 + 2 + 3 + 4
```

Ta muốn parse nó như:

```lox
((1 + 2) + 3) + 4
```

Vì vậy, khi parse toán hạng bên phải của `+` đầu tiên, ta muốn tiêu thụ `2` nhưng không lấy phần còn lại, nên ta dùng mức cao hơn precedence của `+` một bậc. Nhưng nếu toán tử là *right*-associative, điều này sẽ sai. Ví dụ:

```lox
a = b = c = d
```

Vì phép gán là right-associative, ta muốn parse nó như:

```lox
a = (b = (c = d))
```

Để làm được vậy, ta sẽ gọi `parsePrecedence()` với *cùng* mức precedence như toán tử hiện tại.

</aside>

Bằng cách này, ta có thể dùng một hàm `binary()` duy nhất cho tất cả các toán tử nhị phân, dù chúng có precedence khác nhau.

## Pratt Parser

Giờ thì chúng ta đã có đầy đủ các mảnh ghép của compiler. Chúng ta có một hàm cho mỗi production trong grammar: `number()`, `grouping()`, `unary()` và `binary()`. Chúng ta vẫn cần triển khai `parsePrecedence()` và `getRule()`. Chúng ta cũng biết mình cần một bảng mà, khi cho vào một loại token, sẽ cho phép tìm được:

*   Hàm compile một biểu thức prefix bắt đầu bằng token thuộc loại đó.
*   Hàm compile một biểu thức infix mà toán hạng bên trái theo sau bởi token thuộc loại đó.
*   Precedence của một biểu thức <span name="prefix">infix</span> dùng token đó làm toán tử.

<aside name="prefix">

Chúng ta không cần theo dõi precedence của biểu thức *prefix* bắt đầu bằng một token nhất định vì tất cả toán tử prefix trong Lox đều có cùng precedence.

</aside>

Chúng ta gói gọn ba thuộc tính này trong một struct nhỏ, đại diện cho một hàng trong bảng parser.

^code parse-rule (1 before, 2 after)

Kiểu `ParseFn` chỉ là một <span name="typedef">typedef</span> đơn giản cho một kiểu hàm không nhận tham số và không trả về giá trị.

<aside name="typedef" class="bottom">

Cú pháp của C cho kiểu con trỏ hàm tệ đến mức tôi luôn giấu nó sau một typedef. Tôi hiểu ý tưởng đằng sau cú pháp này — kiểu “khai báo phản ánh cách dùng” — nhưng tôi nghĩ đây là một thử nghiệm cú pháp thất bại.

</aside>

^code parse-fn-type (1 before, 2 after)

Bảng điều khiển toàn bộ parser của chúng ta là một mảng `ParseRule`. Chúng ta đã nói về nó mãi rồi, và cuối cùng bạn cũng được thấy nó.

^code rules

<aside name="big">

Bạn thấy tôi nói đúng chưa, tại sao tôi không muốn phải xem lại bảng này mỗi khi cần thêm một cột mới? Nó đúng là một “con quái vật”.

Nếu bạn chưa từng thấy cú pháp `[TOKEN_DOT] = ` trong literal mảng của C, thì đó là cú pháp designated initializer của C99. Nó rõ ràng hơn nhiều so với việc phải đếm chỉ số mảng bằng tay.

</aside>

Bạn có thể thấy `grouping` và `unary` được đặt vào cột parser prefix cho các loại token tương ứng. Ở cột tiếp theo, `binary` được gắn với bốn toán tử số học infix. Các toán tử infix này cũng được thiết lập precedence ở cột cuối.

Ngoài những cái đó, phần còn lại của bảng đầy `NULL` và `PREC_NONE`. Hầu hết các ô trống này là vì không có biểu thức nào gắn với các token đó. Bạn không thể bắt đầu một biểu thức bằng, ví dụ, `else`, và `}` thì sẽ là một toán tử infix khá khó hiểu.

Ngoài ra, chúng ta vẫn chưa hoàn thiện toàn bộ grammar. Ở các chương sau, khi thêm các loại biểu thức mới, một số ô này sẽ được gán hàm. Một điều tôi thích ở cách tiếp cận parsing này là nó giúp dễ dàng thấy token nào đang được grammar sử dụng và token nào còn trống.

Giờ khi đã có bảng, chúng ta cuối cùng cũng sẵn sàng viết code để dùng nó. Đây là lúc Pratt parser của chúng ta “sống” dậy. Hàm dễ định nghĩa nhất là `getRule()`.

^code get-rule

Hàm này đơn giản chỉ trả về rule ở chỉ số được cho. Nó được `binary()` gọi để tra precedence của toán tử hiện tại. Hàm này tồn tại chỉ để xử lý vòng lặp khai báo (declaration cycle) trong code C. `binary()` được định nghĩa *trước* bảng rules để bảng có thể lưu con trỏ tới nó. Điều đó có nghĩa là phần thân của `binary()` không thể truy cập trực tiếp bảng.

Thay vào đó, chúng ta gói thao tác tra cứu vào một hàm. Cách này cho phép chúng ta forward declare `getRule()` trước khi định nghĩa `binary()`, và <span name="forward">sau đó</span> *định nghĩa* `getRule()` sau bảng. Chúng ta sẽ cần thêm một vài forward declaration khác để xử lý việc grammar của chúng ta là đệ quy, nên hãy làm hết chúng luôn.

<aside name="forward">

Đây là điều xảy ra khi bạn viết VM của mình bằng một ngôn ngữ được thiết kế để compile trên PDP-11.

</aside>

^code forward-declarations (2 before, 1 after)

Nếu bạn đang làm theo và tự triển khai clox, hãy chú ý kỹ các ghi chú nhỏ cho biết bạn nên đặt các đoạn code này ở đâu. Nhưng đừng lo, nếu bạn đặt sai, C compiler sẽ vui vẻ báo cho bạn biết.

### Parsing với precedence

Giờ thì chúng ta đến phần thú vị đây. “Nhạc trưởng” điều phối tất cả các hàm parsing mà ta đã định nghĩa chính là `parsePrecedence()`. Hãy bắt đầu với việc parse các biểu thức prefix.

^code precedence-body (1 before, 1 after)

Chúng ta đọc token tiếp theo và tra ParseRule tương ứng. Nếu không có prefix parser, thì token đó chắc chắn là một lỗi cú pháp. Ta báo lỗi và trả về cho hàm gọi.

Ngược lại, ta gọi hàm prefix parser đó và để nó làm phần việc của mình. Prefix parser sẽ compile phần còn lại của biểu thức prefix, tiêu thụ bất kỳ token nào nó cần, rồi trả về đây. Biểu thức infix mới là nơi mọi thứ trở nên thú vị vì precedence bắt đầu đóng vai trò. Việc triển khai lại cực kỳ đơn giản.

^code infix (1 before, 1 after)

Đó là toàn bộ. Thật đấy. Đây là cách toàn bộ hàm hoạt động: Ở đầu `parsePrecedence()`, ta tìm prefix parser cho token hiện tại. Theo định nghĩa, token đầu tiên *luôn* thuộc về một dạng biểu thức prefix nào đó. Nó có thể nằm lồng bên trong như một toán hạng của một hoặc nhiều biểu thức infix, nhưng khi đọc code từ trái sang phải, token đầu tiên bạn gặp luôn thuộc về một biểu thức prefix.

Sau khi parse xong (có thể tiêu thụ thêm nhiều token), biểu thức prefix kết thúc. Giờ ta tìm infix parser cho token tiếp theo. Nếu tìm thấy, điều đó có nghĩa là biểu thức prefix vừa compile có thể là toán hạng của nó. Nhưng chỉ khi lời gọi `parsePrecedence()` có `precedence` đủ thấp để cho phép toán tử infix đó.

Nếu token tiếp theo có precedence quá thấp, hoặc không phải toán tử infix, thì xong — ta đã parse được tối đa phần biểu thức có thể. Ngược lại, ta tiêu thụ toán tử và chuyển quyền điều khiển cho infix parser vừa tìm được. Nó sẽ tiêu thụ các token cần thiết khác (thường là toán hạng bên phải) rồi trả về `parsePrecedence()`. Sau đó, ta lặp lại: kiểm tra xem token *tiếp theo* có phải là toán tử infix hợp lệ có thể lấy toàn bộ biểu thức trước đó làm toán hạng hay không. Ta cứ lặp như vậy, xử lý các toán tử infix và toán hạng của chúng cho đến khi gặp một token không phải infix hoặc có precedence quá thấp thì dừng.

Nghe thì dài dòng, nhưng nếu bạn thật sự muốn “đồng bộ não” với Vaughan Pratt và hiểu trọn vẹn thuật toán, hãy bước qua parser trong debugger khi nó xử lý một vài biểu thức. Có lẽ một bức hình sẽ giúp. Chỉ có vài hàm thôi, nhưng chúng liên kết với nhau cực kỳ chặt chẽ:

<span name="connections"></span>

<img src="image/compiling-expressions/connections.png" alt="Các hàm parsing khác nhau và cách chúng gọi nhau." />

<aside name="connections">

Mũi tên <img src="image/compiling-expressions/calls.png" alt="Mũi tên đặc." class="arrow" /> nối một hàm với hàm khác mà nó gọi trực tiếp.  
Mũi tên <img src="image/compiling-expressions/points-to.png" alt="Mũi tên rỗng." class="arrow" /> thể hiện con trỏ trong bảng trỏ tới các hàm parsing.

</aside>

Sau này, chúng ta sẽ cần chỉnh lại code trong chương này để xử lý assignment. Nhưng ngoài ra, những gì ta đã viết đã bao quát toàn bộ nhu cầu compile biểu thức cho phần còn lại của cuốn sách. Khi thêm các loại biểu thức mới, ta sẽ cắm thêm các hàm parsing vào bảng, nhưng `parsePrecedence()` thì đã hoàn chỉnh.

## Dumping Chunks

Nhân tiện đang ở phần lõi của compiler, ta nên thêm một chút “đo đạc” (instrumentation). Để giúp debug bytecode được generated, ta sẽ thêm khả năng dump chunk sau khi compiler hoàn tất. Trước đây ta có vài log tạm khi tự viết chunk bằng tay. Giờ ta sẽ thêm code thực sự để có thể bật nó bất cứ khi nào muốn.

Vì đây không phải tính năng cho người dùng cuối, ta ẩn nó sau một flag.

^code define-debug-print-code (2 before, 1 after)

Khi flag đó được bật, ta dùng module “debug” sẵn có để in ra bytecode của chunk.

^code dump-chunk (1 before, 1 after)

Ta chỉ làm điều này nếu code không có lỗi. Sau một lỗi cú pháp, compiler vẫn tiếp tục chạy nhưng ở trạng thái hơi “kỳ quặc” và có thể sinh ra code hỏng. Điều đó không gây hại vì code sẽ không được execute, nhưng nếu ta cố đọc nó thì chỉ tự làm mình rối.

Cuối cùng, để truy cập `disassembleChunk()`, ta cần include header của nó.

^code include-debug (1 before, 2 after)

Vậy là xong! Đây là phần lớn cuối cùng cần lắp vào pipeline compile và execute của VM. Interpreter của chúng ta nhìn bề ngoài thì chẳng có gì đặc biệt, nhưng bên trong nó đang scanning, parsing, compile thành bytecode và execute.

Hãy khởi động VM và gõ vào một biểu thức. Nếu mọi thứ đúng, nó sẽ tính toán và in ra kết quả. Giờ ta đã có một “máy tính số học” được thiết kế quá mức cần thiết. Còn rất nhiều tính năng ngôn ngữ sẽ được thêm trong các chương tới, nhưng nền móng đã sẵn sàng.

<div class="challenges">

## Thử thách

1.  Để thật sự hiểu parser, bạn cần thấy luồng execute đi qua các hàm parsing thú vị — `parsePrecedence()` và các hàm parser được lưu trong bảng. Hãy lấy ví dụ biểu thức (hơi kỳ lạ) này:

    ```lox
    (-1 + 2) * 3 - -4
    ```

    Hãy viết một bản trace cho thấy các hàm đó được gọi như thế nào. Thể hiện thứ tự gọi, hàm nào gọi hàm nào, và các tham số được truyền vào.

2.  Hàng ParseRule cho `TOKEN_MINUS` có cả con trỏ hàm prefix và infix. Đó là vì `-` vừa là toán tử prefix (phủ định đơn ngôi) vừa là toán tử infix (phép trừ).

    Trong ngôn ngữ Lox đầy đủ, còn token nào có thể được dùng ở cả vị trí prefix và infix? Còn trong C hoặc một ngôn ngữ khác mà bạn chọn thì sao?

3.  Bạn có thể đang thắc mắc về các biểu thức "mixfix" phức tạp có nhiều hơn hai toán hạng được ngăn cách bởi các token. Toán tử điều kiện hay "ternary" của C, `?:`, là một ví dụ nổi tiếng.

    Hãy thêm hỗ trợ cho toán tử đó vào compiler. Bạn không cần sinh ra bytecode, chỉ cần cho thấy cách bạn sẽ kết nối nó vào parser và xử lý các toán hạng.

</div>

<div class="design-note">

## Ghi chú thiết kế: Chỉ là Parsing thôi mà

Tôi sắp đưa ra một nhận định mà có thể sẽ không được lòng một số người làm compiler và ngôn ngữ. Không sao nếu bạn không đồng ý. Cá nhân tôi học được nhiều hơn từ những ý kiến mạnh mẽ mà tôi không đồng tình so với việc đọc vài trang toàn là điều kiện và rào đón. Nhận định của tôi là *parsing không quan trọng*.

Nhiều năm qua, nhiều người làm ngôn ngữ lập trình, đặc biệt trong giới học thuật, đã *rất* mê mẩn parser và coi nó cực kỳ nghiêm túc. Ban đầu, đó là những người làm compiler đắm chìm trong <span name="yacc">compiler-compilers</span>, LALR, và các thứ tương tự. Nửa đầu của cuốn *dragon book* là một bức thư tình dài gửi đến vẻ đẹp của các parser generator.

<aside name="yacc">

Tất cả chúng ta đều mắc phải “căn bệnh” kiểu “khi tất cả những gì bạn có là một cái búa, mọi thứ trông đều giống cái đinh”, nhưng có lẽ không ai thể hiện rõ điều đó như dân làm compiler. Bạn sẽ không tin được số lượng vấn đề phần mềm mà kỳ diệu thay lại “cần” một ngôn ngữ mới để giải quyết, ngay khi bạn hỏi một compiler hacker.

Yacc và các compiler-compiler khác là ví dụ đệ quy thú vị nhất. “Viết compiler thật là mệt. Biết rồi, hãy viết một compiler để viết compiler cho chúng ta.”

Và để rõ ràng, tôi cũng không miễn nhiễm với “căn bệnh” này.

</aside>

Sau đó, dân lập trình hàm (functional programming) lại mê parser combinator, packrat parser, và các kiểu khác. Vì rõ ràng, nếu bạn đưa cho một lập trình viên hàm một vấn đề, việc đầu tiên họ làm là rút ra một túi đầy các hàm bậc cao.

Bên lĩnh vực toán và phân tích thuật toán, có cả một di sản nghiên cứu lâu dài về việc chứng minh thời gian và bộ nhớ sử dụng cho các kỹ thuật parsing khác nhau, biến đổi bài toán parsing thành các bài toán khác và ngược lại, và gán các lớp độ phức tạp cho các grammar khác nhau.

Ở một mức độ nào đó, những thứ này là quan trọng. Nếu bạn đang triển khai một ngôn ngữ, bạn muốn chắc chắn parser của mình sẽ không rơi vào độ phức tạp hàm mũ và mất 7.000 năm để parse một trường hợp biên kỳ quặc nào đó trong grammar. Lý thuyết parser cho bạn giới hạn đó. Và như một bài tập trí tuệ, học về các kỹ thuật parsing cũng thú vị và bổ ích.

Nhưng nếu mục tiêu của bạn chỉ là triển khai một ngôn ngữ và đưa nó đến tay người dùng, thì gần như tất cả những thứ đó không quan trọng. Rất dễ bị cuốn theo sự hào hứng của những người *thật sự* mê nó và nghĩ rằng front end của bạn *cần* một thứ gì đó hoành tráng kiểu combinator-parser-factory sinh tự động. Tôi đã thấy nhiều người tốn vô số thời gian viết đi viết lại parser của họ bằng bất kỳ thư viện hay kỹ thuật “hot” nào của ngày hôm đó.

Đó là khoảng thời gian không mang lại giá trị gì cho người dùng của bạn. Nếu bạn chỉ muốn parser của mình chạy được, hãy chọn một trong những kỹ thuật tiêu chuẩn, dùng nó, và tiếp tục. Recursive descent, Pratt parsing, và các parser generator phổ biến như ANTLR hoặc Bison đều ổn cả.

Hãy dùng khoảng thời gian bạn tiết kiệm được nhờ không viết lại code parser để cải thiện thông báo lỗi compile mà compiler của bạn hiển thị cho người dùng. Xử lý và báo lỗi tốt có giá trị với người dùng hơn gần như bất kỳ thứ gì khác mà bạn có thể đầu tư thời gian vào ở phần front end.

</div>