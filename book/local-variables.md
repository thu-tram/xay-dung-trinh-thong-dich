
> Và khi trí tưởng tượng hiện hình  
> Những hình thái của điều chưa biết, ngòi bút thi nhân  
> Biến chúng thành hình dạng và trao cho hư vô  
> Một chốn trú ngụ và một cái tên.
>
> <cite>William Shakespeare, <em>Giấc mộng đêm hè</em></cite>

[Chương trước](global-variables.html) đã giới thiệu biến trong clox, nhưng mới chỉ ở dạng <span name="global">global</span>. Trong chương này, ta sẽ mở rộng để hỗ trợ block, block scope và biến local. Trong jlox, ta gói gọn cả phần này và biến global trong một chương. Với clox, đây sẽ là khối lượng công việc của hai chương, một phần vì, thật lòng mà nói, mọi thứ trong C đều tốn nhiều công sức hơn.

<aside name="global">

Chắc hẳn ở đây có thể chêm một câu đùa kiểu “nghĩ toàn cầu, hành động địa phương”, nhưng tôi vẫn chưa nghĩ ra được.

</aside>

Nhưng lý do quan trọng hơn là cách tiếp cận với biến local của ta sẽ khác hẳn so với khi hiện thực biến global. Trong Lox, biến global được **late bound**. “Late” ở đây nghĩa là “được resolve sau khi compile xong”. Điều này tốt cho việc giữ compiler đơn giản, nhưng không tốt cho hiệu năng. Biến local là một trong những <span name="params">thành phần</span> được dùng nhiều nhất trong một ngôn ngữ. Nếu biến local chậm, *mọi thứ* sẽ chậm. Vậy nên ta muốn một chiến lược cho biến local hiệu quả nhất có thể.

<aside name="params">

Tham số hàm cũng được dùng rất nhiều. Chúng hoạt động giống biến local, nên ta sẽ dùng cùng kỹ thuật hiện thực cho chúng.

</aside>

May mắn thay, **lexical scoping** sẽ giúp ta. Như tên gọi, lexical scope nghĩa là ta có thể resolve một biến local chỉ bằng cách nhìn vào văn bản chương trình — biến local *không* phải late bound. Bất kỳ xử lý nào ta làm ở compiler là công việc ta *không* phải làm ở runtime, nên việc hiện thực biến local sẽ tận dụng tối đa compiler.

## Biểu diễn biến Local

Điều tuyệt vời khi “mổ xẻ” một ngôn ngữ lập trình thời nay là ta có cả một lịch sử dài các ngôn ngữ khác để học hỏi. Vậy C và Java quản lý biến local thế nào? Tất nhiên là trên stack! Chúng thường dùng cơ chế stack gốc do chip và hệ điều hành hỗ trợ. Điều đó hơi quá thấp tầng với ta, nhưng trong thế giới ảo của clox, ta có stack riêng để dùng.

Hiện tại, ta mới chỉ dùng nó để giữ **temporaries** — những dữ liệu sống ngắn mà ta cần nhớ trong khi tính toán một biểu thức. Miễn là không cản trở chúng, ta có thể “nhét” biến local vào stack luôn. Điều này rất tốt cho hiệu năng. Cấp phát chỗ cho một biến local mới chỉ cần tăng con trỏ `stackTop`, và giải phóng cũng chỉ cần giảm nó. Truy cập một biến ở vị trí stack đã biết chỉ là một phép truy cập mảng theo chỉ số.

Tuy nhiên, ta cần cẩn thận. VM kỳ vọng stack hoạt động đúng nghĩa “stack”. Ta chỉ được phép cấp phát biến local mới ở đỉnh stack, và chỉ được bỏ một biến local khi không còn gì nằm trên nó. Ngoài ra, cần đảm bảo temporaries không gây cản trở.

May mắn là thiết kế của Lox rất <span name="harmony">ăn khớp</span> với các ràng buộc này. Biến local mới luôn được tạo bởi câu lệnh khai báo. Câu lệnh không lồng bên trong biểu thức, nên sẽ không bao giờ có temporary trên stack khi một câu lệnh bắt đầu execute. Các block được lồng chặt chẽ. Khi một block kết thúc, nó luôn “mang theo” các biến local được khai báo gần nhất, bên trong nhất. Vì đó cũng là những biến vào scope sau cùng, chúng sẽ nằm trên đỉnh stack — đúng vị trí ta cần.

<aside name="harmony">

Sự ăn khớp này rõ ràng không phải ngẫu nhiên. Tôi thiết kế Lox để phù hợp với việc compile một pass sang bytecode dựa trên stack. Nhưng tôi cũng không phải chỉnh sửa ngôn ngữ quá nhiều để đáp ứng các ràng buộc đó. Phần lớn thiết kế của nó sẽ mang lại cảm giác tự nhiên.

Điều này phần lớn là vì lịch sử ngôn ngữ gắn chặt với compile một pass và — ở mức độ thấp hơn — kiến trúc dựa trên stack. Block scope của Lox theo truyền thống kéo dài từ thời BCPL. Là lập trình viên, trực giác của chúng ta về cái gì là “bình thường” trong một ngôn ngữ vẫn bị ảnh hưởng bởi giới hạn phần cứng của những năm xưa.

</aside>

Hãy bước qua ví dụ chương trình này và quan sát cách các biến local xuất hiện và biến mất khỏi scope:

<img src="image/local-variables/scopes.png" alt="Một loạt biến local xuất hiện và biến mất khỏi scope theo kiểu stack." />

Thấy chúng khớp với mô hình stack hoàn hảo chứ? Có vẻ stack sẽ hoạt động tốt để lưu biến local ở runtime. Nhưng ta còn có thể tiến xa hơn. Không chỉ biết *rằng* chúng sẽ nằm trên stack, ta còn có thể xác định chính xác *vị trí* của chúng trên stack. Vì compiler biết chính xác biến local nào đang trong scope tại mọi thời điểm, nó có thể mô phỏng stack trong quá trình compile và ghi chú <span name="fn">vị trí</span> của từng biến trong stack.

Ta sẽ tận dụng điều này bằng cách dùng các stack offset này làm toán hạng cho bytecode instruction đọc và ghi biến local. Điều này khiến việc làm việc với biến local nhanh đến “ngon lành” — đơn giản như truy cập một phần tử mảng.

<aside name="fn">

Trong chương này, biến local bắt đầu từ đáy mảng stack của VM và được đánh chỉ số từ đó. Khi ta thêm [hàm](calls-and-functions.html), sơ đồ này sẽ phức tạp hơn một chút. Mỗi hàm cần vùng stack riêng cho tham số và biến local của nó. Nhưng như ta sẽ thấy, điều đó không làm tăng nhiều độ phức tạp như bạn nghĩ.

</aside>


Có khá nhiều trạng thái mà ta cần phải theo dõi trong compiler để mọi thứ vận hành, nên hãy bắt đầu từ đây. Trong jlox, ta dùng một chuỗi liên kết các HashMap “environment” để theo dõi biến local nào đang nằm trong scope. Đây là cách khá kinh điển, kiểu “sách giáo khoa” để biểu diễn lexical scope. Với clox, như thường lệ, ta sẽ tiến gần hơn tới “kim loại” (low-level). Tất cả trạng thái sẽ nằm trong một struct mới.

^code compiler-struct (1 before, 2 after)

Ta có một mảng phẳng, đơn giản chứa tất cả các biến local đang trong scope tại mỗi thời điểm trong quá trình compile. Chúng được <span name="order">sắp xếp</span> trong mảng theo thứ tự khai báo xuất hiện trong code. Vì toán hạng instruction mà ta dùng để mã hóa một biến local chỉ là một byte, VM của ta có một giới hạn cứng về số lượng biến local có thể cùng nằm trong scope. Điều đó cũng có nghĩa là ta có thể đặt kích thước cố định cho mảng locals.

<aside name="order">

Ta đang viết một compiler một-pass, nên cũng không có *quá* nhiều lựa chọn khác để sắp xếp chúng trong mảng.

</aside>

^code uint8-count (1 before, 2 after)

Quay lại struct Compiler, trường `localCount` theo dõi số lượng biến local đang trong scope — tức là số slot trong mảng đang được dùng. Ta cũng theo dõi “scope depth”. Đây là số lượng block bao quanh đoạn code hiện tại đang compile.

Interpreter Java của ta dùng một chuỗi các map để tách biến của từng block khỏi các block khác. Lần này, ta sẽ đơn giản đánh số biến theo mức độ lồng nhau nơi chúng xuất hiện. Zero là global scope, một là block cấp cao nhất, hai là bên trong nó, bạn hiểu ý rồi đấy. Ta dùng thông tin này để biết mỗi biến local thuộc block nào, từ đó biết biến nào cần bỏ khi block kết thúc.

Mỗi biến local trong mảng là một struct như sau:

^code local-struct (1 before, 2 after)

Ta lưu tên của biến. Khi resolve một identifier, ta so sánh lexeme của identifier với tên của từng biến local để tìm khớp. Khá khó để resolve một biến nếu bạn không biết tên của nó. Trường `depth` ghi lại scope depth của block nơi biến local được khai báo. Đó là tất cả trạng thái ta cần lúc này.

Đây là một cách biểu diễn rất khác so với jlox, nhưng nó vẫn cho phép ta trả lời tất cả các câu hỏi mà compiler cần hỏi về lexical environment. Bước tiếp theo là tìm cách để compiler *truy cập* trạng thái này. Nếu là những kỹ sư <span name="thread">nguyên tắc</span>, ta sẽ truyền một tham số trỏ tới Compiler vào mỗi hàm ở front end. Ta sẽ tạo một Compiler lúc bắt đầu và cẩn thận “luồn” nó qua từng lời gọi hàm… nhưng điều đó sẽ đòi hỏi rất nhiều thay đổi nhàm chán vào code đã viết, nên thay vào đó, ta dùng một biến global:

<aside name="thread">

Đặc biệt, nếu ta muốn dùng compiler trong một ứng dụng đa luồng, có thể với nhiều compiler chạy song song, thì dùng biến global là một ý tưởng *tồi*.

</aside>

^code current-compiler (1 before, 1 after)

Đây là một hàm nhỏ để khởi tạo compiler:

^code init-compiler

Khi VM khởi động lần đầu, ta gọi nó để đưa mọi thứ về trạng thái sạch.

^code compiler (1 before, 1 after)

Compiler của ta đã có dữ liệu cần thiết, nhưng chưa có thao tác trên dữ liệu đó. Chưa có cách để tạo và hủy scope, hoặc thêm và resolve biến. Ta sẽ bổ sung những thứ đó khi cần. Trước hết, hãy bắt đầu xây dựng một số tính năng ngôn ngữ.

## Câu lệnh Block

Trước khi có biến local, ta cần có local scope. Chúng đến từ hai thứ: thân hàm và <span name="block">block</span>. Hàm là một khối lượng công việc lớn mà ta sẽ xử lý trong [một chương sau](functions), nên bây giờ ta chỉ làm block. Như thường lệ, ta bắt đầu với cú pháp. Grammar mới sẽ là:

```ebnf
statement      → exprStmt
               | printStmt
               | block ;

block          → "{" declaration* "}" ;
```

<aside name="block">

Nếu nghĩ kỹ, “block” là một cái tên kỳ lạ. Dùng theo nghĩa ẩn dụ, “block” thường chỉ một đơn vị nhỏ không thể chia, nhưng vì lý do nào đó, ủy ban Algol 60 lại dùng nó để chỉ một cấu trúc *hợp thành* — một chuỗi các câu lệnh. Có thể còn tệ hơn. Algol 58 gọi `begin` và `end` là “dấu ngoặc câu lệnh” (statement parentheses).

<img src="image/local-variables/block.png" alt="Một viên gạch block." class="above" />

</aside>

Block là một loại statement, nên rule cho nó nằm trong production `statement`. Code compile tương ứng trông như sau:

^code parse-block (2 before, 1 after)

Sau khi <span name="helper">parse</span> dấu ngoặc nhọn mở, ta dùng hàm helper này để compile phần còn lại của block:

<aside name="helper">

Hàm này sẽ rất hữu ích sau này khi compile thân hàm.

</aside>

^code block

Hàm này tiếp tục parse các declaration và statement cho đến khi gặp dấu ngoặc nhọn đóng. Như với bất kỳ vòng lặp nào trong parser, ta cũng kiểm tra xem đã hết token chưa. Bằng cách này, nếu chương trình bị lỗi thiếu dấu ngoặc nhọn đóng, compiler sẽ không bị kẹt trong vòng lặp.

Execute một block đơn giản là execute các câu lệnh bên trong nó, lần lượt, nên compile chúng cũng không có gì phức tạp. Điều thú vị về mặt ngữ nghĩa mà block làm là tạo scope. Trước khi compile thân block, ta gọi hàm này để vào một local scope mới:

^code begin-scope

Để “tạo” một scope, tất cả những gì ta làm là tăng depth hiện tại. Cách này chắc chắn nhanh hơn jlox, vốn cấp phát hẳn một HashMap mới cho mỗi scope. Với `beginScope()`, bạn có thể đoán `endScope()` làm gì.

^code end-scope

Vậy là xong phần block và scope — gần như thế — giờ ta đã sẵn sàng để đưa biến vào trong chúng.

## Khai báo biến Local

Thông thường ở đây ta sẽ bắt đầu với parsing, nhưng compiler của ta vốn đã hỗ trợ parse và compile khai báo biến. Ta đã có câu lệnh `var`, biểu thức identifier và gán giá trị. Chỉ là compiler hiện tại giả định tất cả biến đều là global. Vậy nên ta không cần thêm hỗ trợ parsing mới, chỉ cần kết nối ngữ nghĩa scoping mới vào code hiện có.

<img src="image/local-variables/declaration.png" alt="Luồng code bên trong varDeclaration()." />

Quá trình parse khai báo biến bắt đầu trong `varDeclaration()` và dựa vào một vài hàm khác. Đầu tiên, `parseVariable()` tiêu thụ token identifier cho tên biến, thêm lexeme của nó vào constant table của chunk dưới dạng string, rồi trả về chỉ số trong constant table nơi nó được thêm vào. Sau đó, khi `varDeclaration()` compile xong phần initializer, nó gọi `defineVariable()` để phát sinh bytecode lưu giá trị của biến vào bảng băm biến global.

Cả hai helper này cần một vài thay đổi để hỗ trợ biến local. Trong `parseVariable()`, ta thêm:

^code parse-local (1 before, 1 after)

Trước hết, ta “declare” biến. Tôi sẽ giải thích “declare” nghĩa là gì ngay sau đây. Sau đó, nếu đang ở trong local scope, ta thoát khỏi hàm. Ở runtime, biến local không được tra cứu theo tên. Không cần nhét tên biến vào constant table, nên nếu khai báo nằm trong local scope, ta trả về một chỉ số giả trong bảng.

Bên phía `defineVariable()`, nếu đang ở trong local scope, ta cần phát sinh code để lưu biến local. Nó trông như thế này:

^code define-variable (1 before, 1 after)

Khoan, vậy thôi sao? Đúng vậy. Không có code nào để “tạo” biến local ở runtime cả. Hãy nghĩ về trạng thái của VM lúc này: nó đã execute xong code của phần initializer (hoặc `nil` ngầm định nếu người dùng bỏ qua initializer), và giá trị đó đang nằm trên đỉnh stack như temporary duy nhất còn lại. Ta cũng biết biến local mới được cấp phát ở đỉnh stack… đúng vị trí giá trị đó đang ở. Vậy nên chẳng cần làm gì thêm. Temporary đó đơn giản *trở thành* biến local. Khó mà hiệu quả hơn được nữa.

<span name="locals"></span>

<img src="image/local-variables/local-slots.png" alt="Minh họa quá trình execute bytecode cho thấy kết quả của mỗi initializer nằm đúng vào slot của biến local." />

<aside name="locals">

Code bên trái compile thành chuỗi instruction bên phải.

</aside>

OK, vậy “declare” ở đây là gì? Đây là những gì nó làm:

^code declare-variable

Đây là lúc compiler ghi nhận sự tồn tại của biến. Ta chỉ làm điều này cho biến local, nên nếu đang ở global scope cấp cao nhất, ta bỏ qua. Vì biến global là late bound, compiler không theo dõi các khai báo của chúng.

Nhưng với biến local, compiler cần nhớ rằng biến tồn tại. Đó chính là việc “declare” — thêm biến vào danh sách biến trong scope hiện tại của compiler. Ta hiện thực điều này bằng một hàm mới khác:

^code add-local

Hàm này khởi tạo Local tiếp theo còn trống trong mảng biến của compiler. Nó lưu <span name="lexeme">tên</span> biến và độ sâu scope sở hữu biến đó.

<aside name="lexeme">

Lo lắng về vòng đời của string tên biến? Local lưu trực tiếp một bản sao struct Token của identifier. Token lưu con trỏ tới ký tự đầu tiên của lexeme và độ dài lexeme. Con trỏ này trỏ vào string nguồn gốc của script hoặc lệnh REPL đang compile.

Miễn là string đó tồn tại trong suốt quá trình compile — điều chắc chắn vì, bạn biết đấy, ta đang compile nó — thì tất cả token trỏ vào nó đều an toàn.

</aside>

Cách hiện thực này ổn với một chương trình Lox hợp lệ, nhưng còn code không hợp lệ thì sao? Ta nên hướng tới sự “chắc chắn”. Lỗi đầu tiên cần xử lý thực ra không phải lỗi của người dùng, mà là giới hạn của VM. Các instruction làm việc với biến local tham chiếu chúng bằng slot index. Chỉ số này được lưu trong một toán hạng một byte, nghĩa là VM chỉ hỗ trợ tối đa 256 biến local trong scope cùng lúc.

Nếu vượt quá, không chỉ không thể tham chiếu chúng ở runtime, mà compiler còn ghi đè chính mảng locals của nó. Hãy ngăn điều đó:

^code too-many-locals (1 before, 1 after)

Trường hợp tiếp theo phức tạp hơn. Xem ví dụ:

```lox
{
  var a = "first";
  var a = "second";
}
```

Ở cấp cao nhất, Lox cho phép khai báo lại biến cùng tên với một khai báo trước đó vì điều này hữu ích cho REPL. Nhưng bên trong local scope, đó là một điều khá <span name="rust">kỳ quặc</span>. Nhiều khả năng đây là lỗi, và nhiều ngôn ngữ, bao gồm Lox của chúng ta, coi đây là lỗi.

<aside name="rust">

Thú vị là ngôn ngữ Rust *có* cho phép điều này, và code idiomatic của nó dựa vào đó.

</aside>

Lưu ý rằng chương trình trên khác với chương trình này:

```lox
{
  var a = "outer";
  {
    var a = "inner";
  }
}
```

Việc có hai biến cùng tên trong *các scope khác nhau* là OK, ngay cả khi các scope này chồng lấn và cả hai cùng hiển thị tại một thời điểm. Đó là shadowing, và Lox cho phép. Chỉ khi có hai biến cùng tên trong *cùng* một local scope mới là lỗi.

Ta phát hiện lỗi này như sau:

^code existing-in-scope (1 before, 2 after)


<aside name="negative">

Đừng lo về phần `depth != -1` kỳ lạ kia vội. Ta sẽ nói đến nó sau.

</aside>

Biến local được thêm vào cuối mảng khi chúng được khai báo, nghĩa là scope hiện tại luôn nằm ở cuối mảng. Khi ta khai báo một biến mới, ta bắt đầu từ cuối và duyệt ngược lại, tìm một biến đã tồn tại có cùng tên. Nếu tìm thấy một biến trong scope hiện tại, ta báo lỗi. Ngược lại, nếu ta đi đến đầu mảng hoặc gặp một biến thuộc scope khác, thì ta biết mình đã kiểm tra hết tất cả biến trong scope đó.

Để xem hai identifier có giống nhau không, ta dùng hàm sau:

^code identifiers-equal

Vì ta biết độ dài của cả hai lexeme, ta kiểm tra điều đó trước. Điều này sẽ loại nhanh nhiều chuỗi không bằng nhau. Nếu <span name="hash">độ dài</span> giống nhau, ta kiểm tra các ký tự bằng `memcmp()`. Để dùng `memcmp()`, ta cần include thêm.

<aside name="hash">

Sẽ là một tối ưu nhỏ hay ho nếu ta có thể so sánh hash của chúng, nhưng token không phải là LoxString đầy đủ, nên ta chưa tính hash cho chúng.

</aside>

^code compiler-include-string (1 before, 2 after)

Với điều này, ta đã có thể “tạo ra” biến. Nhưng giống như những bóng ma, chúng vẫn “lảng vảng” sau khi scope nơi chúng được khai báo đã kết thúc. Khi một block kết thúc, ta cần “an táng” chúng.

^code pop-locals (1 before, 1 after)

Khi ta pop một scope, ta duyệt ngược qua mảng local để tìm bất kỳ biến nào được khai báo ở scope depth vừa rời khỏi. Ta loại bỏ chúng đơn giản bằng cách giảm độ dài mảng.

Có một phần liên quan đến runtime ở đây. Biến local chiếm các slot trên stack. Khi một biến local ra khỏi scope, slot đó không còn cần thiết và nên được giải phóng. Vì vậy, với mỗi biến bị loại bỏ, ta cũng phát sinh một lệnh `OP_POP` <span name="pop"></span> để pop nó khỏi stack.

<aside name="pop">

Khi nhiều biến local ra khỏi scope cùng lúc, bạn sẽ có một loạt lệnh `OP_POP` được execute lần lượt. Một tối ưu đơn giản bạn có thể thêm vào Lox của mình là một lệnh chuyên biệt `OP_POPN` nhận một toán hạng cho số slot cần pop và pop tất cả cùng lúc.

</aside>

## Sử dụng biến Local

Giờ ta đã có thể compile và execute khai báo biến local. Ở runtime, giá trị của chúng nằm đúng vị trí trên stack. Hãy bắt đầu sử dụng chúng. Ta sẽ làm cả truy cập và gán biến cùng lúc vì chúng dùng chung các hàm trong compiler.

Ta đã có code để lấy và gán biến global, và — như những kỹ sư phần mềm tốt bụng — ta muốn tái sử dụng càng nhiều code hiện có càng tốt. Đại loại như thế này:

^code named-local (1 before, 2 after)

Thay vì hardcode các bytecode instruction được phát sinh cho việc truy cập và gán biến, ta dùng một vài biến C. Đầu tiên, ta thử tìm một biến local với tên đã cho. Nếu tìm thấy, ta dùng instruction dành cho biến local. Nếu không, ta giả định đó là biến global và dùng instruction bytecode hiện có cho biến global.

Ngay bên dưới, ta dùng các biến đó để phát sinh instruction phù hợp. Với gán giá trị:

^code emit-set (2 before, 1 after)

Và với truy cập:

^code emit-get (2 before, 1 after)

Phần cốt lõi của chương này, nơi ta resolve một biến local, nằm ở đây:

^code resolve-local

Nhìn chung, nó khá đơn giản. Ta duyệt danh sách các biến local hiện đang trong scope. Nếu một biến có cùng tên với token identifier, thì identifier đó chắc chắn tham chiếu tới biến này. Ta đã tìm thấy nó! Ta duyệt mảng ngược lại để tìm biến được khai báo *sau cùng* với identifier đó. Điều này đảm bảo rằng biến local bên trong sẽ shadow đúng cách biến local cùng tên ở scope bao ngoài.

Ở runtime, ta load và store biến local bằng chỉ số slot trên stack, nên đó là thứ compiler cần tính toán sau khi resolve biến. Mỗi khi một biến được khai báo, ta thêm nó vào mảng locals trong Compiler. Điều đó có nghĩa là biến local đầu tiên ở index 0, biến tiếp theo ở index 1, và cứ thế. Nói cách khác, mảng locals trong compiler có bố cục *chính xác* như stack của VM ở runtime. Chỉ số của biến trong mảng locals cũng chính là slot của nó trên stack. Thật tiện lợi!

Nếu ta duyệt hết mảng mà không tìm thấy biến có tên đã cho, thì chắc chắn nó không phải biến local. Trong trường hợp đó, ta trả về `-1` để báo rằng không tìm thấy và giả định nó là biến global.


### Execute biến local

Compiler của ta đang phát sinh hai instruction mới, nên hãy khiến chúng hoạt động. Đầu tiên là load một biến local:

^code get-local-op (1 before, 1 after)

Và phần hiện thực của nó:

^code interpret-get-local (1 before, 1 after)

Nó nhận một toán hạng một byte cho slot trên stack nơi biến local nằm. Nó lấy giá trị từ chỉ số đó rồi push nó lên đỉnh stack để các instruction tiếp theo có thể tìm thấy.

<aside name="slot">

Có vẻ như việc push giá trị của biến local lên stack là dư thừa vì nó vốn đã nằm đâu đó thấp hơn trên stack. Vấn đề là các bytecode instruction khác chỉ tìm dữ liệu ở *đỉnh* stack. Đây là đặc điểm cốt lõi khiến tập lệnh bytecode của ta là dạng *stack*-based. Tập lệnh bytecode [register-based](a-virtual-machine.html#design-note) tránh được việc “xoay” stack này, nhưng phải trả giá bằng các instruction lớn hơn với nhiều toán hạng hơn.

</aside>

Tiếp theo là gán giá trị:

^code set-local-op (1 before, 1 after)

Chắc bạn cũng đoán được phần hiện thực.

^code interpret-set-local (1 before, 1 after)

Nó lấy giá trị được gán từ đỉnh stack và lưu vào slot trên stack tương ứng với biến local. Lưu ý rằng nó không pop giá trị đó khỏi stack. Hãy nhớ rằng, gán là một biểu thức, và mọi biểu thức đều tạo ra một giá trị. Giá trị của một biểu thức gán chính là giá trị được gán, nên VM chỉ việc để nguyên giá trị đó trên stack.

Disassembler của ta sẽ chưa hoàn chỉnh nếu không hỗ trợ hai instruction mới này.

^code disassemble-local (1 before, 1 after)

Compiler compile biến local thành truy cập trực tiếp slot. Tên của biến local không bao giờ rời khỏi compiler để xuất hiện trong chunk. Điều này rất tốt cho hiệu năng, nhưng không tốt cho việc introspection. Khi disassemble các instruction này, ta không thể hiển thị tên biến như với biến global. Thay vào đó, ta chỉ hiển thị số slot.

<aside name="debug">

Việc xóa tên biến local trong compiler sẽ là một vấn đề thực sự nếu sau này ta muốn hiện thực debugger cho VM. Khi người dùng bước qua code, họ mong muốn thấy giá trị của các biến local được sắp xếp theo tên. Để hỗ trợ điều đó, ta sẽ cần xuất thêm thông tin bổ sung để theo dõi tên của từng biến local tại mỗi slot trên stack.

</aside>

^code byte-instruction

### Một edge case khác về scope

Ta đã dành thời gian xử lý một vài edge case kỳ lạ quanh scope. Ta đảm bảo shadowing hoạt động đúng. Ta báo lỗi nếu hai biến trong cùng một local scope có cùng tên. Vì lý do nào đó mà tôi cũng không hoàn toàn rõ, scoping của biến dường như luôn có nhiều “nếp nhăn” kiểu này. Tôi chưa từng thấy ngôn ngữ nào mà nó cảm giác hoàn toàn <span name="elegant">mượt mà</span>.

<aside name="elegant">

Không, kể cả Scheme.

</aside>

Ta còn một edge case nữa cần xử lý trước khi kết thúc chương này. Hãy nhớ lại “con quái vật” kỳ lạ mà ta từng gặp trong [phần hiện thực variable resolution của jlox](resolving-and-binding.html#resolving-variable-declarations):

```lox
{
  var a = "outer";
  {
    var a = a;
  }
}
```

Ta đã “hạ gục” nó khi đó bằng cách tách khai báo biến thành hai giai đoạn, và ta sẽ làm lại điều đó ở đây:

<img src="image/local-variables/phases.png" alt="Ví dụ khai báo biến được đánh dấu 'declared uninitialized' trước tên biến và 'ready for use' sau phần initializer." />

Ngay khi khai báo biến bắt đầu — tức là trước phần initializer — tên biến được khai báo trong scope hiện tại. Biến tồn tại, nhưng ở trạng thái đặc biệt “chưa khởi tạo”. Sau đó ta compile phần initializer. Nếu ở bất kỳ điểm nào trong biểu thức đó, ta resolve một identifier trỏ lại biến này, ta sẽ thấy nó chưa được khởi tạo và báo lỗi. Sau khi compile xong initializer, ta đánh dấu biến là đã khởi tạo và sẵn sàng sử dụng.

Để hiện thực điều này, khi khai báo một biến local, ta cần biểu thị trạng thái “chưa khởi tạo” bằng cách nào đó. Ta có thể thêm một trường mới vào Local, nhưng hãy tiết kiệm bộ nhớ hơn một chút. Thay vào đó, ta sẽ đặt scope depth của biến thành một giá trị sentinel đặc biệt, `-1`.

^code declare-undefined (1 before, 1 after)

Sau đó, khi phần initializer của biến đã được compile, ta đánh dấu nó là đã khởi tạo.

^code define-local (1 before, 2 after)

Phần này được hiện thực như sau:

^code mark-initialized

Vậy đây mới *thực sự* là ý nghĩa của “declare” và “define” một biến trong compiler. “Declare” là khi biến được thêm vào scope, và “define” là khi nó sẵn sàng để sử dụng.

Khi ta resolve một tham chiếu tới biến local, ta kiểm tra scope depth để xem nó đã được định nghĩa đầy đủ chưa.

^code own-initializer-error (1 before, 1 after)

Nếu biến có depth là giá trị sentinel, thì chắc chắn đó là một tham chiếu tới biến trong chính initializer của nó, và ta báo lỗi.

Vậy là xong chương này! Ta đã thêm block, biến local, và lexical scoping “xịn”. Mặc dù ta đã giới thiệu một cách biểu diễn biến ở runtime hoàn toàn khác, nhưng lượng code phải viết không nhiều. Phần hiện thực cuối cùng khá gọn gàng và hiệu quả.

Bạn sẽ nhận thấy gần như toàn bộ code ta viết nằm ở compiler. Còn ở runtime, chỉ có hai instruction nhỏ. Bạn sẽ thấy đây là một <span name="static">xu hướng</span> tiếp diễn trong clox so với jlox. Một trong những “vũ khí” lớn nhất trong hộp công cụ của optimizer là kéo công việc về phía compiler để không phải làm ở runtime. Trong chương này, điều đó có nghĩa là resolve chính xác slot trên stack mà mỗi biến local chiếm. Nhờ vậy, ở runtime, không cần lookup hay resolve gì nữa.

<aside name="static">

Bạn có thể xem static type như một ví dụ cực đoan của xu hướng này. Một ngôn ngữ có kiểu tĩnh sẽ thực hiện toàn bộ việc phân tích kiểu và xử lý lỗi kiểu ngay trong quá trình biên dịch. Nhờ đó, runtime không phải tốn thời gian kiểm tra xem giá trị có đúng kiểu cho phép với phép toán hay không. Thực tế, trong một số ngôn ngữ kiểu tĩnh như C, bạn thậm chí còn *không biết* kiểu của giá trị ở runtime. Compiler sẽ xóa hoàn toàn mọi thông tin biểu diễn kiểu của giá trị, chỉ để lại phần bit thuần túy.

</aside>

<div class="challenges">

## Thử thách

1.  Mảng local đơn giản của ta giúp việc tính toán stack slot của mỗi biến local trở nên dễ dàng. Nhưng điều đó cũng có nghĩa là khi compiler resolve một tham chiếu tới biến, ta phải quét tuyến tính qua mảng.

    Hãy nghĩ ra một cách hiệu quả hơn. Bạn có cho rằng độ phức tạp tăng thêm là xứng đáng không?

2.  Các ngôn ngữ khác xử lý đoạn code như thế này thế nào:

    ```lox
    var a = a;
    ```

    Nếu đây là ngôn ngữ của bạn, bạn sẽ làm gì? Tại sao?

3.  Nhiều ngôn ngữ phân biệt giữa biến có thể gán lại và biến không thể gán lại. Trong Java, từ khóa `final` ngăn bạn gán lại cho biến. Trong JavaScript, biến khai báo bằng `let` có thể gán lại, nhưng biến khai báo bằng `const` thì không. Swift coi `let` là biến chỉ gán một lần và dùng `var` cho biến có thể gán lại. Scala và Kotlin dùng `val` và `var`.

    Hãy chọn một từ khóa cho dạng biến chỉ gán một lần để thêm vào Lox. Giải thích lý do chọn, rồi hiện thực nó. Nếu cố gán cho một biến được khai báo bằng từ khóa mới này, compiler phải báo lỗi.

4.  Mở rộng clox để cho phép nhiều hơn 256 biến local cùng nằm trong scope tại một thời điểm.

</div>