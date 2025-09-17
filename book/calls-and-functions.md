> Bất kỳ vấn đề nào trong khoa học máy tính cũng có thể được giải quyết bằng một tầng gián tiếp khác. Ngoại trừ vấn đề có quá nhiều tầng gián tiếp.
>
> <cite>David Wheeler</cite>

Chương này đúng là một “con quái vật”. Tôi thường cố gắng chia nhỏ tính năng thành từng phần dễ nuốt, nhưng đôi khi bạn phải nuốt trọn cả <span name="eat">bữa</span>. Nhiệm vụ tiếp theo của chúng ta là functions. Ta có thể bắt đầu chỉ với khai báo function, nhưng điều đó chẳng hữu ích mấy khi bạn không thể gọi chúng. Ta có thể làm phần gọi hàm, nhưng lại chẳng có gì để gọi. Và tất cả phần hỗ trợ runtime trong VM để phục vụ cả hai thứ đó cũng chẳng đem lại nhiều giá trị nếu nó không kết nối với thứ gì bạn có thể thấy. Vậy nên, chúng ta sẽ làm tất cả. Khá nhiều việc, nhưng khi xong, bạn sẽ thấy rất đáng.

<aside name="eat">

“Ăn” — tiêu thụ — là một phép ẩn dụ kỳ lạ cho một hành động sáng tạo. Nhưng hầu hết các quá trình sinh học tạo ra “đầu ra” đều, ừm, kém phần tao nhã hơn một chút.

</aside>

## Function Objects

Thay đổi cấu trúc thú vị nhất trong VM lần này nằm ở stack. Chúng ta đã _có_ một stack cho biến cục bộ và giá trị tạm thời, nên coi như đã đi được nửa đường. Nhưng ta chưa có khái niệm _call_ stack. Trước khi tiến xa hơn, ta sẽ phải xử lý điều đó. Nhưng trước hết, hãy viết chút code. Tôi luôn thấy dễ chịu hơn khi bắt đầu hành động. Chúng ta không thể làm được nhiều nếu chưa có cách biểu diễn function, nên hãy bắt đầu từ đó. Từ góc nhìn của VM, function là gì?

Một function có phần thân có thể execute, tức là có bytecode. Ta có thể compile toàn bộ chương trình và tất cả khai báo function vào một Chunk khổng lồ duy nhất. Mỗi function sẽ có một con trỏ trỏ tới instruction đầu tiên của code của nó bên trong Chunk.

Đây cũng gần giống cách compile sang native code, nơi bạn nhận được một khối machine code nguyên khối. Nhưng với bytecode VM của chúng ta, ta có thể làm ở mức trừu tượng cao hơn một chút. Tôi nghĩ mô hình gọn gàng hơn là cho mỗi function một Chunk riêng. Ta cũng sẽ cần thêm một số metadata khác, nên hãy nhét tất cả vào một struct ngay bây giờ.

^code obj-function (2 before, 2 after)

Functions là first-class trong Lox, nên chúng cần là các object Lox thực sự. Vì vậy, ObjFunction có cùng phần header Obj mà tất cả các loại object khác chia sẻ. Trường `arity` lưu số lượng tham số mà function mong đợi. Sau đó, ngoài chunk, ta lưu cả <span name="name">tên</span> của function. Điều này sẽ hữu ích khi báo lỗi runtime dễ đọc.

<aside name="name">

Con người thường không thấy các bytecode offset dạng số mang lại nhiều thông tin trong crash dump.

</aside>

Đây là lần đầu tiên module “object” cần tham chiếu tới Chunk, nên ta thêm một include.

^code object-include-chunk (1 before, 1 after)

Giống như với string, ta định nghĩa một số hàm tiện ích để làm việc với function Lox trong C dễ hơn. Kiểu như một phiên bản “lập trình hướng đối tượng” đơn giản. Đầu tiên, ta khai báo một hàm C để tạo một function Lox mới.

^code new-function-h (3 before, 1 after)

Phần hiện thực nằm ở đây:

^code new-function

Ta dùng `ALLOCATE_OBJ()` để cấp phát bộ nhớ và khởi tạo header của object để VM biết nó thuộc loại object nào. Thay vì truyền tham số để khởi tạo function như với ObjString, ta thiết lập function ở trạng thái “trắng” — arity bằng 0, không tên, không code. Những thông tin này sẽ được điền sau khi function được tạo.

Vì có một loại object mới, ta cần thêm một object type mới vào enum.

^code obj-type-function (1 before, 2 after)

Khi xong việc với một function object, ta phải trả lại phần bộ nhớ mà nó đã mượn từ hệ điều hành.

^code free-function (1 before, 1 after)

Case trong switch này <span name="free-name">chịu trách nhiệm</span> giải phóng chính ObjFunction cũng như bất kỳ vùng nhớ nào nó sở hữu. Functions sở hữu chunk của chúng, nên ta gọi hàm “giống destructor” của Chunk.

<aside name="free-name">

Ta không cần giải phóng tên của function một cách tường minh vì nó là một ObjString. Điều đó có nghĩa là ta có thể để garbage collector quản lý vòng đời của nó. Hoặc ít nhất, ta sẽ làm được điều đó khi [hiện thực garbage collector](garbage-collection.html).

</aside>

Lox cho phép bạn in bất kỳ object nào, và functions là first-class objects, nên ta cũng cần xử lý chúng.

^code print-function (1 before, 1 after)

Hàm này gọi tới:

^code print-function-helper

Vì function biết tên của mình, nên nó có thể tự “giới thiệu”.

Cuối cùng, ta có một vài macro để chuyển đổi value thành function. Đầu tiên, đảm bảo value của bạn thực sự _là_ một function.

^code is-function (2 before, 1 after)

Nếu điều đó đúng, bạn có thể an toàn cast Value thành con trỏ ObjFunction bằng cách dùng:

^code as-function (2 before, 1 after)

Vậy là mô hình object của chúng ta đã biết cách biểu diễn functions. Tôi đang nóng máy rồi đấy. Sẵn sàng cho phần khó hơn chưa?

## Biên dịch sang Function Object

Hiện tại, compiler của chúng ta giả định rằng nó luôn biên dịch vào một chunk duy nhất. Khi mỗi function có code riêng nằm trong các chunk riêng biệt, mọi thứ sẽ phức tạp hơn. Khi compiler gặp một khai báo function, nó cần phát sinh code vào chunk của function đó khi biên dịch phần thân. Cuối phần thân function, compiler cần quay lại chunk trước đó mà nó đang làm việc.

Điều này ổn với code bên trong thân function, nhưng còn code không nằm trong đó thì sao? “Top level” của một chương trình Lox cũng là code dạng mệnh lệnh, và chúng ta cần một chunk để biên dịch nó. Ta có thể đơn giản hóa compiler và VM bằng cách đặt cả code top-level này vào bên trong một function được định nghĩa tự động. Bằng cách đó, compiler luôn ở trong một thân function nào đó, và VM luôn chạy code bằng cách gọi một function. Giống như toàn bộ chương trình được <span name="wrap">bọc</span> bên trong một `main()` ngầm định.

<aside name="wrap">

Một điểm khác biệt về semantics khiến phép so sánh này không hoàn toàn đúng là biến toàn cục. Chúng có quy tắc phạm vi đặc biệt khác với biến cục bộ, nên theo cách đó, top level của một script không giống thân function.

</aside>

Vậy nên, trước khi đến với function do người dùng định nghĩa, hãy thực hiện việc tái cấu trúc để hỗ trợ function top-level ngầm định này. Bắt đầu với struct Compiler. Thay vì trỏ trực tiếp tới một Chunk mà compiler sẽ ghi code vào, nó sẽ giữ tham chiếu tới function object đang được xây dựng.

^code function-fields (1 before, 1 after)

Chúng ta cũng có một enum FunctionType nhỏ. Nó cho phép compiler biết khi nào đang biên dịch code top-level và khi nào đang biên dịch thân function. Phần lớn compiler không quan tâm đến điều này — đó là lý do tại sao đây là một lớp trừu tượng hữu ích — nhưng ở một vài chỗ, sự phân biệt này lại quan trọng. Chúng ta sẽ gặp một ví dụ sau.

^code function-type-enum

Mọi chỗ trong compiler trước đây ghi vào Chunk giờ cần thông qua con trỏ `function`. May mắn là từ <span name="current">nhiều chương</span> trước, chúng ta đã đóng gói việc truy cập chunk vào hàm `currentChunk()`. Chỉ cần sửa ở đó là phần còn lại của compiler sẽ ổn.

<aside name="current">

Gần như thể tôi có một quả cầu pha lê nhìn thấy trước tương lai và biết rằng sau này sẽ cần thay đổi code. Nhưng thực ra là vì tôi đã viết toàn bộ code cho cuốn sách trước khi viết phần chữ.

</aside>

^code current-chunk (1 before, 2 after)

Chunk hiện tại luôn là chunk thuộc về function mà ta đang biên dịch. Tiếp theo, ta cần thực sự tạo function đó. Trước đây, VM truyền một Chunk cho compiler để nó điền code vào. Giờ thì compiler sẽ tạo và trả về một function chứa code đã compile của top-level — vốn là tất cả những gì chúng ta hỗ trợ hiện tại — của chương trình người dùng.

### Tạo function khi compile

Chúng ta bắt đầu luồng này trong `compile()`, hàm entry point chính của compiler.

^code call-init-compiler (1 before, 2 after)

Có khá nhiều thay đổi trong cách khởi tạo compiler. Đầu tiên, ta khởi tạo các trường mới của Compiler.

^code init-compiler (1 after)

Sau đó, ta cấp phát một function object mới để compile vào đó.

^code init-function (1 before, 1 after)

<span name="null"></span>

<aside name="null">

Tôi biết, việc gán `null` cho trường `function` rồi ngay sau đó lại gán giá trị cho nó vài dòng sau trông hơi ngớ ngẩn. Đây là một chút “ám ảnh” liên quan đến garbage collection.

</aside>

Việc tạo một ObjFunction trong compiler có thể trông hơi lạ. Function object là biểu diễn _runtime_ của một function, nhưng ở đây ta lại tạo nó khi compile. Cách nghĩ là: function giống như một literal string hoặc number. Nó tạo ra một cầu nối giữa thế giới compile-time và runtime. Khi chúng ta đến với _khai báo_ function, chúng thực sự _là_ literal — một dạng ký pháp tạo ra giá trị của một kiểu dựng sẵn. Vì vậy, <span name="closure">compiler</span> tạo function object trong quá trình compile. Sau đó, ở runtime, chúng chỉ đơn giản được gọi.

<aside name="closure">

Chúng ta có thể tạo function khi compile vì chúng chỉ chứa dữ liệu có sẵn tại compile-time. Code, tên và arity của function đều cố định. Khi chúng ta thêm closures trong [chương tiếp theo](closures.html), vốn bắt biến tại runtime, câu chuyện sẽ phức tạp hơn.

</aside>

Đây là một đoạn code kỳ lạ khác:

^code init-function-slot (1 before, 1 after)

Hãy nhớ rằng mảng `locals` của compiler dùng để theo dõi stack slot nào gắn với biến cục bộ hoặc biến tạm nào. Từ giờ trở đi, compiler sẽ ngầm “giữ chỗ” stack slot số 0 cho mục đích nội bộ của VM. Chúng ta gán cho nó một tên rỗng để người dùng không thể viết một identifier tham chiếu tới nó. Tôi sẽ giải thích điều này khi nó trở nên hữu ích.

Đó là phần khởi tạo. Chúng ta cũng cần một vài thay đổi ở phía kết thúc khi compile xong một đoạn code.

^code end-compiler (1 after)

Trước đây, khi `interpret()` gọi vào compiler, nó truyền vào một Chunk để ghi code. Giờ thì compiler tự tạo function object, nên chúng ta sẽ trả về function đó. Ta lấy nó từ compiler hiện tại ở đây:

^code end-function (1 before, 1 after)

Và trả nó về cho `compile()` như sau:

^code return-function (1 before, 1 after)

Đây cũng là lúc thích hợp để chỉnh sửa thêm một chút trong hàm này. Trước đó, chúng ta đã thêm code chẩn đoán để VM dump bytecode đã disassemble nhằm debug compiler. Giờ ta cần sửa lại để nó vẫn hoạt động khi chunk được tạo ra nằm bên trong một function.

^code disassemble-end (2 before, 2 after)

Hãy chú ý đoạn kiểm tra xem tên của function có phải `NULL` không. Function do người dùng định nghĩa thì có tên, nhưng function ngầm định mà chúng ta tạo cho code top-level thì không, và ta cần xử lý điều đó một cách gọn gàng ngay cả trong code chẩn đoán của chính mình. Nói về chuyện này:

^code print-script (1 before, 1 after)

Người _dùng_ không thể lấy tham chiếu tới function top-level và thử in nó, nhưng code chẩn đoán `DEBUG_TRACE_EXECUTION` <span name="debug">in toàn bộ stack</span> thì có thể và thực sự làm điều đó.

<aside name="debug">

Sẽ chẳng vui vẻ gì nếu code chẩn đoán mà ta dùng để tìm bug lại chính là nguyên nhân khiến VM segfault!

</aside>

Lên một tầng tới `compile()`, ta chỉnh lại signature hàm.

^code compile-h (2 before, 2 after)

Thay vì nhận một chunk, giờ nó trả về một function. Trong phần hiện thực:

^code compile-signature (1 after)

Cuối cùng, ta đến với một chút code thực sự. Ta thay đổi phần cuối của hàm thành:

^code call-end-compiler (4 before, 1 after)

Ta lấy function object từ compiler. Nếu không có lỗi compile, ta trả về nó. Ngược lại, ta báo lỗi bằng cách trả về `NULL`. Cách này giúp VM không cố execute một function có thể chứa bytecode không hợp lệ.

Cuối cùng, chúng ta sẽ cập nhật `interpret()` để xử lý khai báo mới của `compile()`, nhưng trước đó còn một số thay đổi khác cần thực hiện.

## Call Frames

Đã đến lúc thực hiện một bước nhảy khái niệm lớn. Trước khi có thể hiện thực khai báo và lời gọi function, chúng ta cần chuẩn bị cho VM sẵn sàng xử lý chúng. Có hai vấn đề chính cần quan tâm:

### Cấp phát biến cục bộ

Compiler cấp phát stack slot cho các biến cục bộ. Vậy điều đó sẽ hoạt động thế nào khi tập hợp biến cục bộ trong chương trình được phân tán qua nhiều function?

Một lựa chọn là giữ chúng hoàn toàn tách biệt. Mỗi function sẽ có một tập slot riêng trong stack của VM mà nó sở hữu <span name="static">vĩnh viễn</span>, ngay cả khi function đó không được gọi. Mỗi biến cục bộ trong toàn bộ chương trình sẽ có một vùng nhớ riêng trong VM mà nó giữ cho mình.

<aside name="static">

Về cơ bản, đây giống như việc bạn khai báo mọi biến cục bộ trong một chương trình C bằng từ khóa `static`.

</aside>

Tin hay không thì tùy, nhưng các implementation ngôn ngữ lập trình thời kỳ đầu đã từng làm như vậy. Những compiler Fortran đầu tiên cấp phát bộ nhớ tĩnh cho từng biến. Vấn đề hiển nhiên là điều này cực kỳ kém hiệu quả. Phần lớn function không ở trạng thái đang được gọi tại bất kỳ thời điểm nào, nên việc giữ bộ nhớ không dùng cho chúng là lãng phí.

Vấn đề cơ bản hơn là đệ quy. Với đệ quy, bạn có thể “đang ở trong” nhiều lời gọi tới cùng một function cùng lúc. Mỗi lời gọi cần <span name="fortran">vùng nhớ riêng</span> cho các biến cục bộ của nó. Trong jlox, chúng ta giải quyết điều này bằng cách cấp phát động bộ nhớ cho một environment mỗi khi function được gọi hoặc khi vào một block. Trong clox, chúng ta không muốn chịu chi phí hiệu năng đó ở mỗi lần gọi function.

<aside name="fortran">

Fortran tránh vấn đề này bằng cách cấm hoàn toàn đệ quy. Thời đó, đệ quy được coi là một tính năng nâng cao, mang tính “huyền thuật”.

</aside>

Thay vào đó, giải pháp của chúng ta nằm đâu đó giữa cách cấp phát tĩnh của Fortran và cách tiếp cận động của jlox. Value stack trong VM dựa trên quan sát rằng các biến cục bộ và biến tạm thời có hành vi theo kiểu last-in first-out. May mắn cho chúng ta, điều này vẫn đúng ngay cả khi bạn thêm lời gọi hàm vào. Đây là một ví dụ:

```lox
fun first() {
  var a = 1;
  second();
  var b = 2;
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

Hãy đi từng bước qua chương trình và xem tại mỗi thời điểm, biến nào đang nằm trong bộ nhớ:

<img src="image/calls-and-functions/calls.png" alt="Tracing through the execution of the previous program, showing the stack of variables at each step." />

Khi luồng execute đi qua hai lời gọi hàm, mọi biến cục bộ đều tuân theo nguyên tắc: bất kỳ biến nào được khai báo sau nó sẽ bị loại bỏ trước khi biến đầu tiên cần được loại bỏ. Điều này đúng ngay cả giữa các lời gọi hàm. Ta biết rằng ta sẽ xong việc với `c` và `d` trước khi xong với `a`. Có vẻ như ta hoàn toàn có thể cấp phát biến cục bộ trên value stack của VM.

Lý tưởng nhất, ta vẫn muốn xác định _vị trí_ của mỗi biến trên stack ngay tại compile time. Điều đó giúp các instruction bytecode làm việc với biến trở nên đơn giản và nhanh chóng. Trong ví dụ trên, ta có thể <span name="imagine">hình dung</span> việc này một cách đơn giản, nhưng không phải lúc nào cũng làm được. Xem thử ví dụ sau:

<aside name="imagine">

Tôi nói “hình dung” vì compiler thực ra không thể xác định được điều này. Vì functions là first-class trong Lox, ta không thể biết được hàm nào sẽ gọi hàm nào tại compile time.

</aside>

```lox
fun first() {
  var a = 1;
  second();
  var b = 2;
  second();
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

Trong lần gọi đầu tiên tới `second()`, `c` và `d` sẽ nằm ở slot 1 và 2. Nhưng trong lần gọi thứ hai, ta cần chừa chỗ cho `b`, nên `c` và `d` phải nằm ở slot 2 và 3. Do đó, compiler không thể cố định một slot chính xác cho mỗi biến cục bộ xuyên suốt các lời gọi hàm. Nhưng _bên trong_ một hàm nhất định, vị trí _tương đối_ của mỗi biến cục bộ là cố định. Biến `d` luôn nằm ngay sau `c`. Đây chính là điểm mấu chốt.

Khi một hàm được gọi, ta không biết đỉnh stack sẽ ở đâu vì nó có thể được gọi từ nhiều ngữ cảnh khác nhau. Nhưng, bất kể đỉnh stack ở đâu, ta vẫn biết tất cả biến cục bộ của hàm sẽ nằm ở đâu so với điểm bắt đầu đó. Vậy nên, giống như nhiều vấn đề khác, ta giải quyết bài toán cấp phát này bằng một tầng gián tiếp.

Khi bắt đầu mỗi lời gọi hàm, VM sẽ ghi lại vị trí của slot đầu tiên nơi các biến cục bộ của hàm đó bắt đầu. Các instruction làm việc với biến cục bộ sẽ truy cập chúng bằng chỉ số slot _tương đối_ với vị trí này, thay vì tương đối với đáy stack như hiện tại. Tại compile time, ta tính toán các slot tương đối đó. Tại runtime, ta chuyển slot tương đối thành chỉ số tuyệt đối trên stack bằng cách cộng thêm vị trí bắt đầu của lời gọi hàm.

Cứ như thể hàm có một “cửa sổ” hoặc “frame” bên trong stack lớn hơn, nơi nó có thể lưu các biến cục bộ của mình. Vị trí của **call frame** được xác định tại runtime, nhưng bên trong và tương đối với vùng đó, ta biết chính xác mọi thứ nằm ở đâu.

<img src="image/calls-and-functions/window.png" alt="The stack at the two points when second() is called, with a window hovering over each one showing the pair of stack slots used by the function." />

Tên gọi lịch sử cho vị trí được ghi lại này — nơi các biến cục bộ của hàm bắt đầu — là **frame pointer** vì nó trỏ tới đầu call frame của hàm. Đôi khi bạn sẽ nghe thấy **base pointer**, vì nó trỏ tới slot gốc của stack, nơi tất cả biến của hàm được đặt lên.

Đó là mảnh dữ liệu đầu tiên mà ta cần theo dõi. Mỗi khi gọi một hàm, VM sẽ xác định slot đầu tiên trên stack nơi các biến của hàm đó bắt đầu.

### Return addresses

Hiện tại, VM duyệt qua luồng instruction bằng cách tăng trường `ip`. Hành vi thú vị duy nhất là ở các instruction điều khiển luồng, vốn thay đổi `ip` với một giá trị offset lớn hơn. _Gọi_ một hàm thì khá đơn giản — chỉ cần đặt `ip` trỏ tới instruction đầu tiên trong chunk của hàm đó. Nhưng khi hàm chạy xong thì sao?

VM cần <span name="return">quay lại</span> chunk nơi hàm được gọi và tiếp tục execute tại instruction ngay sau lời gọi. Do đó, với mỗi lời gọi hàm, ta cần theo dõi vị trí sẽ nhảy về khi lời gọi kết thúc. Đây được gọi là **return address** vì nó là địa chỉ của instruction mà VM sẽ quay lại sau khi gọi xong.

Và một lần nữa, nhờ có đệ quy, có thể tồn tại nhiều return address cho cùng một hàm, nên đây là thuộc tính của từng _lần gọi_ chứ không phải của bản thân hàm.

<aside name="return">

Các tác giả của những compiler Fortran đầu tiên đã có một mẹo khá thông minh để hiện thực return address. Vì họ _không_ hỗ trợ đệ quy, nên mỗi hàm chỉ cần một return address duy nhất tại bất kỳ thời điểm nào. Vậy nên, khi một hàm được gọi tại runtime, chương trình sẽ _tự sửa code của mình_ để thay đổi một lệnh jump ở cuối hàm, nhảy về vị trí của caller. Đôi khi, ranh giới giữa thiên tài và điên rồ thật mong manh.

</aside>

### The call stack

Vậy là, với mỗi lần gọi hàm đang hoạt động — tức là mỗi lời gọi chưa trả về — chúng ta cần theo dõi vị trí trên stack nơi các biến cục bộ của hàm đó bắt đầu, và vị trí mà caller sẽ tiếp tục execute. Chúng ta sẽ đặt thông tin này, cùng với một số dữ liệu khác, vào một struct mới.

^code call-frame (1 before, 2 after)

Một CallFrame đại diện cho một lần gọi hàm đang diễn ra. Trường `slots` trỏ vào value stack của VM tại slot đầu tiên mà hàm này có thể sử dụng. Tôi đặt tên ở dạng số nhiều vì — nhờ vào “đặc sản” của C là “con trỏ gần như là mảng” — chúng ta sẽ xử lý nó như một mảng.

Cách hiện thực return address hơi khác một chút so với mô tả trước đó. Thay vì lưu return address trong frame của callee, caller sẽ lưu `ip` của chính nó. Khi trả về từ một hàm, VM sẽ nhảy tới `ip` của CallFrame của caller và tiếp tục từ đó.

Tôi cũng nhét vào đây một con trỏ tới function đang được gọi. Chúng ta sẽ dùng nó để tra cứu constants và cho một vài việc khác.

Mỗi khi một hàm được gọi, chúng ta tạo một struct như thế này. Chúng ta _có thể_ <span name="heap">cấp phát động</span> chúng trên heap, nhưng như vậy sẽ chậm. Lời gọi hàm là một thao tác cốt lõi, nên cần nhanh nhất có thể. May mắn là, ta có thể áp dụng cùng một quan sát như với biến: lời gọi hàm có tính chất stack. Nếu `first()` gọi `second()`, thì lời gọi tới `second()` sẽ hoàn tất trước khi `first()` hoàn tất.

<aside name="heap">

Nhiều implementation của Lisp cấp phát động stack frame vì nó giúp đơn giản hóa việc hiện thực [continuations](https://en.wikipedia.org/wiki/Continuation). Nếu ngôn ngữ của bạn hỗ trợ continuations, thì lời gọi hàm _không_ phải lúc nào cũng có tính chất stack.

</aside>

Vì vậy, trong VM, chúng ta tạo sẵn một mảng các CallFrame và xử lý nó như một stack, giống như cách ta làm với mảng value.

^code frame-array (1 before, 1 after)

Mảng này thay thế cho các trường `chunk` và `ip` mà trước đây chúng ta lưu trực tiếp trong VM. Giờ đây, mỗi CallFrame có `ip` riêng và con trỏ riêng tới ObjFunction mà nó đang execute. Từ đó, ta có thể truy cập chunk của function.

Trường `frameCount` mới trong VM lưu chiều cao hiện tại của CallFrame stack — tức là số lượng lời gọi hàm đang diễn ra. Để giữ cho clox đơn giản, dung lượng của mảng này là cố định. Điều này có nghĩa là, giống như nhiều implementation ngôn ngữ khác, có một độ sâu gọi hàm tối đa mà chúng ta có thể xử lý. Với clox, nó được định nghĩa ở đây:

^code frame-max (2 before, 2 after)

Chúng ta cũng định nghĩa lại <span name="plenty">kích thước</span> của value stack dựa trên giá trị này để đảm bảo có đủ stack slot ngay cả trong những cây lời gọi rất sâu. Khi VM khởi động, CallFrame stack sẽ rỗng.

<aside name="plenty">

Vẫn có thể xảy ra tràn stack nếu đủ nhiều lời gọi hàm sử dụng đủ nhiều biến tạm ngoài biến cục bộ. Một implementation “xịn” sẽ có cơ chế bảo vệ chống lại điều này, nhưng tôi đang cố giữ mọi thứ đơn giản.

</aside>

^code reset-frame-count (1 before, 1 after)

Header "vm.h" cần truy cập ObjFunction, nên ta thêm một include.

^code vm-include-object (2 before, 1 after)

Giờ chúng ta sẵn sàng chuyển sang file hiện thực của VM. Sẽ có kha khá việc “dọn dẹp” ở đây. Chúng ta đã chuyển `ip` ra khỏi struct VM và đưa vào CallFrame. Ta cần sửa mọi dòng code trong VM có đụng tới `ip` để xử lý điều đó. Ngoài ra, các instruction truy cập biến cục bộ theo stack slot cũng cần được cập nhật để làm việc tương đối với trường `slots` của CallFrame hiện tại.

Chúng ta sẽ bắt đầu từ trên xuống và xử lý lần lượt.

^code run (1 before, 1 after)

Đầu tiên, ta lưu CallFrame trên cùng hiện tại vào một biến <span name="local">local</span> bên trong hàm execute bytecode chính. Sau đó, ta thay thế các macro truy cập bytecode bằng các phiên bản truy cập `ip` thông qua biến đó.

<aside name="local">

Chúng ta _có thể_ truy cập frame hiện tại bằng cách đi qua mảng CallFrame mỗi lần, nhưng như vậy sẽ dài dòng. Quan trọng hơn, lưu frame vào một biến local sẽ khuyến khích C compiler giữ con trỏ đó trong một thanh ghi. Điều này giúp tăng tốc độ truy cập `ip` của frame. Không có _đảm bảo_ rằng compiler sẽ làm vậy, nhưng khả năng cao là nó sẽ.

</aside>

Giờ chúng ta sẽ đến với từng instruction cần được “chăm sóc” một chút.

^code push-local (2 before, 1 after)

Trước đây, `OP_GET_LOCAL` đọc slot local được chỉ định trực tiếp từ mảng stack của VM, nghĩa là nó đánh chỉ số slot tính từ đáy stack. Giờ đây, nó truy cập mảng `slots` của frame hiện tại, tức là truy cập slot theo số thứ tự _tương đối_ so với điểm bắt đầu của frame đó.

Việc gán giá trị cho một biến local cũng hoạt động tương tự.

^code set-local (2 before, 1 after)

Các instruction nhảy (jump) trước đây thay đổi trường `ip` của VM. Giờ thì chúng cũng làm điều tương tự nhưng với `ip` của frame hiện tại.

^code jump (2 before, 1 after)

Tương tự với jump có điều kiện:

^code jump-if-false (2 before, 1 after)

Và instruction vòng lặp nhảy lùi của chúng ta:

^code loop (2 before, 1 after)

Chúng ta có một đoạn code chẩn đoán in ra từng instruction khi nó được execute để giúp debug VM. Đoạn này cũng cần hoạt động với cấu trúc mới.

^code trace-execution (1 before, 1 after)

Thay vì truyền vào `chunk` và `ip` của VM, giờ ta đọc từ CallFrame hiện tại.

Bạn thấy đấy, thực ra cũng không quá khó. Hầu hết các instruction chỉ dùng macro nên không cần chỉnh sửa. Tiếp theo, ta lên một tầng tới đoạn code gọi `run()`.

^code interpret-stub (1 before, 2 after)

Cuối cùng, chúng ta kết nối các thay đổi ở compiler trước đó với các thay đổi ở back-end vừa thực hiện. Đầu tiên, ta truyền source code vào compiler. Nó trả về một ObjFunction mới chứa code top-level đã compile. Nếu nhận về `NULL`, nghĩa là có lỗi compile-time và compiler đã báo lỗi. Trong trường hợp đó, ta dừng lại vì không thể chạy gì được.

Ngược lại, ta lưu function này lên stack và chuẩn bị một CallFrame ban đầu để execute code của nó. Giờ bạn sẽ thấy vì sao compiler dành riêng stack slot số 0 — đó là nơi lưu function đang được gọi. Trong CallFrame mới, ta trỏ tới function, khởi tạo `ip` của nó trỏ tới đầu bytecode của function, và thiết lập “cửa sổ” stack của nó bắt đầu từ đáy value stack của VM.

Điều này giúp interpreter sẵn sàng bắt đầu execute code. Sau khi chạy xong, trước đây VM sẽ giải phóng chunk được hardcode. Giờ ObjFunction sở hữu code đó, nên ta không cần làm vậy nữa. Kết thúc `interpret()` giờ chỉ đơn giản là:

^code end-interpret (2 before, 1 after)

Phần code cuối cùng còn tham chiếu tới các trường cũ của VM là `runtimeError()`. Chúng ta sẽ quay lại nó sau trong chương này, nhưng tạm thời sửa thành:

^code runtime-error-temp (2 before, 1 after)

Thay vì đọc chunk và `ip` trực tiếp từ VM, nó lấy từ CallFrame trên cùng của stack. Như vậy function sẽ hoạt động trở lại và hành xử như trước.

Nếu mọi thứ được làm đúng, clox đã trở lại trạng thái chạy được. Chạy thử và nó… vẫn làm đúng như trước. Chúng ta chưa thêm tính năng mới nào, nên hơi hụt hẫng. Nhưng toàn bộ hạ tầng đã sẵn sàng. Giờ hãy tận dụng nó.

## Function Declarations

Trước khi có thể làm call expression, ta cần có cái để gọi, nên sẽ làm function declaration trước. Từ khóa <span name="fun">fun</span> mở đầu cho phần này.

<aside name="fun">

Vâng, tôi sẽ pha trò ngớ ngẩn về từ khóa `fun` mỗi khi nó xuất hiện.

</aside>

^code match-fun (1 before, 1 after)

Điều này chuyển quyền điều khiển tới đây:

^code fun-declaration

Functions là giá trị first-class, và một function declaration đơn giản là tạo và lưu nó vào một biến mới khai báo. Vậy nên ta parse tên giống như bất kỳ khai báo biến nào khác. Function declaration ở top-level sẽ gán function cho một biến global. Bên trong block hoặc function khác, function declaration sẽ tạo biến local.

Ở một chương trước, tôi đã giải thích cách biến [được định nghĩa qua hai giai đoạn](local-variables.html#another-scope-edge-case). Điều này đảm bảo bạn không thể truy cập giá trị của biến ngay trong phần khởi tạo của chính nó. Điều đó sẽ tệ vì biến chưa _có_ giá trị.

Functions không gặp vấn đề này. Việc một function tham chiếu tới tên của chính nó bên trong thân là an toàn. Bạn không thể _gọi_ function và execute thân của nó cho tới khi nó được định nghĩa đầy đủ, nên bạn sẽ không bao giờ thấy biến ở trạng thái chưa khởi tạo. Trên thực tế, việc cho phép điều này rất hữu ích để hỗ trợ các hàm local đệ quy.

Để làm được điều đó, ta đánh dấu biến của function declaration là “initialized” ngay khi compile tên, trước khi compile thân. Như vậy tên có thể được tham chiếu bên trong thân mà không gây lỗi.

Tuy nhiên, ta cần một bước kiểm tra:

^code check-depth (1 before, 1 after)

Trước đây, ta chỉ gọi `markInitialized()` khi đã biết mình đang ở trong local scope. Giờ, function declaration ở top-level cũng sẽ gọi hàm này. Khi đó, không có biến local nào để đánh dấu — function được gán cho một biến global.

Tiếp theo, ta compile chính function — danh sách tham số và block body. Để làm điều đó, ta dùng một helper riêng. Helper này sinh code để để lại function object kết quả trên đỉnh stack. Sau đó, ta gọi `defineVariable()` để lưu function đó vào biến đã khai báo.

Tôi tách riêng code compile tham số và thân vì sau này sẽ tái sử dụng khi parse method declaration trong class. Hãy xây dựng dần dần, bắt đầu với:

^code compile-function

<aside name="no-end-scope">

`beginScope()` này không có lời gọi `endScope()` tương ứng. Vì chúng ta kết thúc Compiler hoàn toàn khi tới cuối thân function, nên không cần đóng scope ngoài cùng còn lại.

</aside>

Hiện tại, ta chưa quan tâm tới tham số. Ta parse một cặp ngoặc đơn rỗng, sau đó là thân hàm. Thân bắt đầu bằng dấu ngoặc nhọn mở, ta parse ở đây. Sau đó, ta gọi hàm `block()` sẵn có, vốn biết cách compile phần còn lại của block bao gồm cả dấu ngoặc nhọn đóng.

### A stack of compilers

Phần thú vị nằm ở chỗ xử lý compiler ở đầu và cuối. Struct Compiler lưu trữ dữ liệu như slot nào thuộc về biến cục bộ nào, chúng ta đang ở mức lồng nhau bao nhiêu block, v.v. Tất cả những thứ đó đều gắn với một function duy nhất. Nhưng giờ front end cần xử lý việc compile nhiều function <span name="nested">lồng nhau</span> bên trong nhau.

<aside name="nested">

Hãy nhớ rằng compiler coi code top-level như thân của một function ngầm định, nên ngay khi ta thêm _bất kỳ_ function declaration nào, ta đã bước vào thế giới của các function lồng nhau.

</aside>

Mẹo để quản lý việc này là tạo một Compiler riêng cho mỗi function đang được compile. Khi bắt đầu compile một function declaration, ta tạo một Compiler mới trên C stack và khởi tạo nó. `initCompiler()` đặt Compiler đó thành compiler hiện tại. Sau đó, khi compile thân hàm, tất cả các hàm sinh bytecode sẽ ghi vào chunk thuộc về function của Compiler mới này.

Khi tới cuối block body của function, ta gọi `endCompiler()`. Hàm này trả về function object vừa compile xong, và ta lưu nó như một constant trong bảng constant của function _bao quanh_ nó. Nhưng khoan, làm sao ta quay lại function bao quanh? Ta đã mất nó khi `initCompiler()` ghi đè con trỏ compiler hiện tại.

Ta giải quyết bằng cách coi chuỗi các Compiler lồng nhau như một stack. Khác với Value và CallFrame stack trong VM, ta sẽ không dùng mảng. Thay vào đó, ta dùng linked list. Mỗi Compiler trỏ ngược về Compiler của function bao quanh nó, cho tới Compiler gốc của code top-level.

^code enclosing-field (2 before, 1 after)

Bên trong struct Compiler, ta không thể tham chiếu tới _typedef_ Compiler vì khai báo đó chưa hoàn tất. Thay vào đó, ta đặt tên cho chính struct và dùng tên đó cho kiểu của field. C đúng là kỳ quặc.

Khi khởi tạo một Compiler mới, ta lưu compiler sắp không còn là hiện tại nữa vào con trỏ đó.

^code store-enclosing (1 before, 1 after)

Rồi khi một Compiler kết thúc, nó “pop” chính nó khỏi stack bằng cách khôi phục compiler trước đó thành compiler hiện tại.

^code restore-enclosing (2 before, 1 after)

Lưu ý rằng ta thậm chí không cần <span name="compiler">cấp phát động</span> các struct Compiler. Mỗi cái được lưu như một biến local trên C stack — hoặc trong `compile()` hoặc `function()`. Linked list các Compiler được “xâu chuỗi” qua C stack. Lý do ta có thể có số lượng không giới hạn là vì compiler của chúng ta dùng recursive descent, nên `function()` sẽ tự gọi lại chính nó khi có các function declaration lồng nhau.

<aside name="compiler">

Việc dùng native stack cho các struct Compiler đồng nghĩa compiler của ta có một giới hạn thực tế về độ sâu lồng nhau của function declaration. Nếu đi quá xa, bạn có thể làm tràn C stack. Nếu muốn compiler “trâu bò” hơn trước các code bất thường hoặc thậm chí độc hại — điều này là mối quan tâm thực sự với các công cụ như JavaScript VM — thì sẽ tốt hơn nếu compiler tự giới hạn độ sâu function nesting mà nó cho phép.

</aside>

### Function parameters

Function sẽ chẳng hữu ích mấy nếu bạn không thể truyền tham số cho chúng, nên giờ ta sẽ làm phần parameters.

^code parameters (1 before, 1 after)

Về mặt ngữ nghĩa, một parameter đơn giản là một biến cục bộ được khai báo trong phạm vi từ vựng ngoài cùng của thân hàm. Ta có thể tận dụng cơ chế sẵn có của compiler để khai báo biến cục bộ có tên nhằm parse và compile parameters. Khác với biến cục bộ, vốn có initializer, ở đây không có code để khởi tạo giá trị cho parameter. Chúng ta sẽ thấy chúng được khởi tạo thế nào sau khi làm phần truyền đối số trong lời gọi hàm.

Nhân tiện, ta ghi nhận arity của function bằng cách đếm số parameter parse được. Thông tin metadata khác mà ta lưu cùng function là tên của nó. Khi compile một function declaration, ta gọi `initCompiler()` ngay sau khi parse tên hàm. Điều đó có nghĩa là ta có thể lấy tên ngay lúc đó từ token trước đó.

^code init-function-name (1 before, 2 after)

Lưu ý rằng ta cẩn thận tạo một bản sao của chuỗi tên. Hãy nhớ rằng lexeme trỏ trực tiếp vào chuỗi source code gốc. Chuỗi đó có thể bị giải phóng khi compile xong. Function object mà ta tạo trong compiler tồn tại lâu hơn compiler và kéo dài tới runtime. Vì vậy nó cần một chuỗi tên được cấp phát trên heap riêng để giữ lại.

Tuyệt. Giờ ta có thể compile function declaration, như thế này:

```lox
fun areWeHavingItYet() {
  print "Yes we are!";
}

print areWeHavingItYet;
```

Chỉ là ta chưa thể làm gì <span name="useful">hữu ích</span> với chúng.

<aside name="useful">

Ta có thể in chúng! Nhưng chắc là cũng chẳng hữu ích lắm.

</aside>

## Function Calls

Kết thúc phần này, chúng ta sẽ bắt đầu thấy một số hành vi thú vị. Bước tiếp theo là gọi hàm. Thường thì ta không nghĩ theo cách này, nhưng một biểu thức gọi hàm thực chất giống như một toán tử `(` dạng infix. Bên trái là một biểu thức có độ ưu tiên cao — thường chỉ là một identifier — biểu thị thứ sẽ được gọi. Tiếp theo là dấu `(` ở giữa, sau đó là các biểu thức đối số (argument) được phân tách bằng dấu phẩy, và cuối cùng là dấu `)` để kết thúc.

Cách nhìn ngữ pháp hơi lạ này giải thích cách chúng ta gắn cú pháp vào bảng phân tích (parsing table).

^code infix-left-paren (1 before, 1 after)

Khi parser gặp dấu ngoặc đơn mở ngay sau một biểu thức, nó sẽ chuyển sang một hàm parser mới.

^code compile-call

Chúng ta đã đọc xong token `(`, nên bước tiếp theo là compile các đối số bằng một helper riêng `argumentList()`. Hàm này trả về số lượng đối số mà nó compile được. Mỗi biểu thức đối số sẽ sinh code để lại giá trị của nó trên stack, sẵn sàng cho lời gọi hàm. Sau đó, ta phát sinh một instruction `OP_CALL` mới để gọi hàm, dùng số lượng đối số làm toán hạng (operand).

Chúng ta compile các đối số bằng “người bạn” này:

^code argument-list

Đoạn code này chắc sẽ trông quen thuộc từ jlox. Ta xử lý các đối số miễn là còn tìm thấy dấu phẩy sau mỗi biểu thức. Khi hết, ta đọc dấu ngoặc đơn đóng cuối cùng và xong.

À, gần xong thôi. Trong jlox, ta đã thêm một kiểm tra ở compile-time để đảm bảo bạn không truyền quá 255 đối số cho một lời gọi. Lúc đó, tôi đã nói rằng vì clox cũng sẽ cần một giới hạn tương tự. Giờ bạn có thể thấy lý do — vì chúng ta nhét số lượng đối số vào bytecode như một toán hạng 1 byte, nên chỉ có thể tối đa 255. Ta cũng cần xác minh điều này trong compiler này.

^code arg-limit (1 before, 1 after)

Đó là phần front end. Giờ hãy chuyển sang back end, với một điểm dừng nhanh ở giữa để khai báo instruction mới.

^code op-call (1 before, 1 after)

### Binding arguments to parameters

Trước khi đi vào hiện thực, ta nên nghĩ về trạng thái stack tại thời điểm gọi hàm và những gì cần làm từ đó. Khi tới instruction gọi hàm, chúng ta đã execute xong biểu thức cho hàm được gọi, tiếp theo là các đối số của nó. Giả sử chương trình của ta như sau:

```lox
fun sum(a, b, c) {
  return a + b + c;
}

print 4 + sum(5, 6, 7);
```

Nếu ta tạm dừng VM ngay tại instruction `OP_CALL` cho lời gọi `sum()`, stack sẽ trông như thế này:

<img src="image/calls-and-functions/argument-stack.png" alt="Stack: 4, fn sum, 5, 6, 7." />

Hãy hình dung từ góc nhìn của chính `sum()`. Khi compiler compile `sum()`, nó tự động cấp phát slot số 0. Sau đó, nó cấp phát các slot local cho các tham số `a`, `b` và `c`, theo thứ tự. Để thực hiện lời gọi tới `sum()`, ta cần một CallFrame được khởi tạo với hàm được gọi và một vùng slot stack mà nó có thể dùng. Sau đó, ta cần thu thập các đối số truyền vào hàm và đặt chúng vào đúng slot tương ứng với các tham số.

Khi VM bắt đầu execute thân của `sum()`, ta muốn “cửa sổ” stack của nó trông như thế này:

<img src="image/calls-and-functions/parameter-window.png" alt="The same stack with the sum() function's call frame window surrounding fn sum, 5, 6, and 7." />

Bạn có nhận ra rằng các slot đối số mà caller thiết lập và các slot tham số mà callee cần đều ở đúng thứ tự không? Thật tiện lợi! Đây không phải là ngẫu nhiên. Khi tôi nói về việc mỗi CallFrame có “cửa sổ” riêng vào stack, tôi chưa bao giờ nói rằng các cửa sổ đó phải _tách biệt_. Không có gì ngăn cản chúng ta cho chúng chồng lấn lên nhau, như thế này:

<img src="image/calls-and-functions/overlapping-windows.png" alt="The same stack with the top-level call frame covering the entire stack and the sum() function's call frame window surrounding fn sum, 5, 6, and 7." />

<span name="lua">Phần</span> trên cùng của stack của caller chứa hàm được gọi, theo sau là các đối số theo thứ tự. Ta biết caller không có slot nào khác phía trên đang được dùng, vì mọi biến tạm cần thiết khi đánh giá các biểu thức đối số đã bị loại bỏ. Phần đáy của stack của callee chồng lấn sao cho các slot tham số khớp chính xác với vị trí các giá trị đối số đã có sẵn.

<aside name="lua">

Các bytecode VM và kiến trúc CPU thực tế khác nhau có các _calling convention_ khác nhau — tức là cơ chế cụ thể mà chúng dùng để truyền đối số, lưu return address, v.v. Cơ chế tôi dùng ở đây dựa trên virtual machine gọn gàng và nhanh của Lua.

</aside>

Điều này có nghĩa là ta không cần làm _bất kỳ_ công việc nào để “gắn đối số vào tham số”. Không có việc sao chép giá trị giữa các slot hay qua các environment. Các đối số đã ở đúng vị trí cần thiết. Khó mà tìm được cách nào nhanh hơn thế.

Đến lúc hiện thực instruction gọi hàm.

^code interpret-call (1 before, 1 after)

Ta cần biết hàm được gọi và số lượng đối số truyền vào. Ta lấy số lượng đối số từ toán hạng của instruction. Thông tin này cũng cho ta biết vị trí của hàm trên stack bằng cách đếm ngược từ đỉnh stack qua các slot đối số. Ta chuyển dữ liệu đó cho hàm `callValue()` riêng. Nếu hàm này trả về `false`, nghĩa là lời gọi gây ra lỗi runtime. Khi đó, ta dừng interpreter.

Nếu `callValue()` thành công, sẽ có một frame mới trên CallFrame stack cho hàm được gọi. Hàm `run()` có một con trỏ cache tới frame hiện tại, nên ta cần cập nhật nó.

^code update-frame-after-call (2 before, 1 after)

Vì vòng lặp dispatch bytecode đọc từ biến `frame` đó, khi VM execute instruction tiếp theo, nó sẽ đọc `ip` từ CallFrame của hàm vừa được gọi và nhảy tới code của nó. Công việc execute lời gọi bắt đầu ở đây:

^code call-value

<aside name="switch">

Dùng câu lệnh `switch` để kiểm tra một kiểu duy nhất thì hơi “quá tay” bây giờ, nhưng sẽ hợp lý khi ta thêm các trường hợp xử lý những kiểu có thể gọi khác.

</aside>

Ở đây có nhiều việc hơn là chỉ khởi tạo một CallFrame mới. Vì Lox là ngôn ngữ _dynamically typed_, không có gì ngăn người dùng viết code tệ kiểu như:

```lox
var notAFunction = 123;
notAFunction();
```

Nếu điều đó xảy ra, runtime cần báo lỗi một cách an toàn và dừng lại. Vậy nên việc đầu tiên ta làm là kiểm tra kiểu của giá trị mà ta đang cố gọi. Nếu nó không phải là function, ta báo lỗi và thoát. Ngược lại, lời gọi thực sự diễn ra ở đây:

^code call

Phần này chỉ đơn giản là khởi tạo CallFrame tiếp theo trên stack. Nó lưu một con trỏ tới function đang được gọi và đặt `ip` của frame trỏ tới đầu bytecode của function. Cuối cùng, nó thiết lập con trỏ `slots` để cấp cho frame “cửa sổ” của nó vào stack. Phép tính ở đây đảm bảo rằng các đối số đã có trên stack sẽ thẳng hàng với các tham số của function:

<img src="image/calls-and-functions/arithmetic.png" alt="The arithmetic to calculate frame-&gt;slots from stackTop and argCount." />

Dấu `- 1` nhỏ kia là để tính tới stack slot số 0 mà compiler đã dành riêng cho việc thêm method sau này. Các tham số bắt đầu ở slot số 1, nên ta cho cửa sổ bắt đầu sớm hơn một slot để chúng thẳng hàng với các đối số.

Trước khi tiếp tục, hãy thêm instruction mới này vào disassembler.

^code disassemble-call (1 before, 1 after)

Và thêm một bước nhỏ nữa. Giờ khi ta đã có một hàm tiện lợi để khởi tạo CallFrame, ta cũng có thể dùng nó để thiết lập frame đầu tiên cho việc execute code top-level.

^code interpret (1 before, 2 after)

OK, quay lại với phần gọi hàm...

### Runtime error checking

Cơ chế chồng lấn các “cửa sổ” stack hoạt động dựa trên giả định rằng một lời gọi truyền chính xác một đối số cho mỗi tham số của function. Nhưng, một lần nữa, vì Lox không phải _statically typed_, một người dùng bất cẩn có thể truyền quá nhiều hoặc quá ít đối số. Trong Lox, ta định nghĩa đây là một lỗi runtime, và báo lỗi như sau:

^code check-arity (1 before, 1 after)

Khá đơn giản. Đây là lý do ta lưu arity của mỗi function bên trong ObjFunction của nó.

Có một lỗi khác cần báo, lỗi này ít liên quan tới sự bất cẩn của người dùng hơn là của chính chúng ta. Vì mảng CallFrame có kích thước cố định, ta cần đảm bảo một chuỗi lời gọi quá sâu không làm tràn nó.

^code check-overflow (2 before, 1 after)

Trên thực tế, nếu một chương trình tiến gần tới giới hạn này, rất có thể là do bug trong một đoạn code đệ quy mất kiểm soát.

### Printing stack traces

Nhân nói về runtime error, hãy dành chút thời gian để làm cho chúng hữu ích hơn. Dừng lại khi gặp runtime error là quan trọng để ngăn VM “nổ tung” theo một cách khó lường. Nhưng chỉ đơn giản thoát ra thì không giúp người dùng sửa code _gây ra_ lỗi đó.

Công cụ kinh điển để hỗ trợ debug lỗi runtime là **stack trace** — bản in ra của từng function vẫn đang execute khi chương trình chết, và vị trí execute tại thời điểm đó. Giờ khi ta đã có call stack và tiện lợi lưu tên của từng function, ta có thể hiển thị toàn bộ stack này khi một runtime error phá vỡ “sự yên bình” của người dùng. Nó trông như thế này:

^code runtime-error-stack (2 before, 2 after)

<aside name="minus">

Dấu `- 1` là vì IP đã trỏ tới instruction tiếp theo sẽ được execute, nhưng ta muốn stack trace chỉ tới instruction lỗi trước đó.

</aside>

Sau khi in thông báo lỗi, ta duyệt call stack từ <span name="top">trên xuống</span> (function được gọi gần nhất) tới dưới cùng (code top-level). Với mỗi frame, ta tìm số dòng tương ứng với `ip` hiện tại bên trong function của frame đó. Sau đó, ta in số dòng cùng với tên function.

<aside name="top">

Có một chút tranh luận về việc nên hiển thị stack frame theo thứ tự nào. Hầu hết đặt function trong cùng lên đầu và đi dần xuống đáy stack. Python thì in theo thứ tự ngược lại. Đọc từ trên xuống sẽ cho bạn biết chương trình đã đi tới chỗ hiện tại như thế nào, và dòng cuối cùng là nơi lỗi thực sự xảy ra.

Cách đó cũng có lý. Nó đảm bảo bạn luôn thấy function trong cùng ngay cả khi stack trace quá dài để hiển thị hết trên một màn hình. Mặt khác, nguyên tắc “[inverted pyramid](<https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism)>)” trong báo chí nói rằng ta nên đặt thông tin quan trọng nhất _lên đầu_ trong một khối văn bản. Trong stack trace, đó là function nơi lỗi thực sự xảy ra. Hầu hết các implementation ngôn ngữ khác làm theo cách này.

</aside>

Ví dụ, nếu bạn chạy chương trình lỗi này:

```lox
fun a() { b(); }
fun b() { c(); }
fun c() {
  c("too", "many");
}

a();
```

Nó sẽ in ra:

```text
Expected 0 arguments but got 2.
[line 4] in c()
[line 2] in b()
[line 1] in a()
[line 7] in script
```

Trông cũng không tệ lắm, đúng không?

### Returning from functions

Chúng ta sắp hoàn thiện rồi. Giờ ta đã có thể gọi hàm, và VM sẽ execute chúng. Nhưng ta vẫn chưa thể _return_ từ chúng. Chúng ta đã có instruction `OP_RETURN` từ khá lâu, nhưng nó luôn chứa một đoạn code tạm thời chỉ để thoát khỏi vòng lặp bytecode. Giờ là lúc hiện thực nó một cách “đúng nghĩa”.

^code interpret-return (1 before, 1 after)

Khi một hàm trả về một giá trị, giá trị đó sẽ nằm trên đỉnh stack. Chúng ta sắp loại bỏ toàn bộ “cửa sổ” stack của hàm được gọi, nên ta pop giá trị trả về đó ra và giữ lại. Sau đó, ta loại bỏ CallFrame của hàm vừa trả về. Nếu đó là CallFrame cuối cùng, nghĩa là ta đã execute xong code top-level. Toàn bộ chương trình đã hoàn tất, nên ta pop hàm script chính ra khỏi stack và thoát interpreter.

Ngược lại, nếu vẫn còn frame khác, ta loại bỏ tất cả các slot mà callee đã dùng cho tham số và biến cục bộ. Điều này bao gồm cả các slot mà caller đã dùng để truyền đối số. Giờ khi lời gọi đã xong, caller không cần chúng nữa. Điều này có nghĩa là đỉnh stack sẽ nằm ngay tại vị trí bắt đầu của “cửa sổ” stack của hàm vừa trả về.

Ta push giá trị trả về lại lên stack ở vị trí mới, thấp hơn đó. Sau đó, ta cập nhật con trỏ cache của hàm `run()` tới frame hiện tại. Giống như khi bắt đầu một lời gọi, ở vòng lặp dispatch bytecode tiếp theo, VM sẽ đọc `ip` từ frame đó, và việc execute sẽ nhảy trở lại caller, ngay tại vị trí nó dừng, ngay sau instruction `OP_CALL`.

<img src="image/calls-and-functions/return.png" alt="Each step of the return process: popping the return value, discarding the call frame, pushing the return value." />

Lưu ý rằng ở đây ta giả định hàm _thực sự_ trả về một giá trị, nhưng một hàm có thể ngầm trả về bằng cách chạy tới cuối thân hàm:

```lox
fun noReturn() {
  print "Do stuff";
  // Không có return ở đây.
}

print noReturn(); // ???
```

Ta cũng cần xử lý đúng trường hợp này. Ngôn ngữ được định nghĩa là sẽ ngầm trả về `nil` trong trường hợp đó. Để làm điều này, ta thêm đoạn sau:

^code return-nil (1 before, 2 after)

Compiler sẽ gọi `emitReturn()` để ghi instruction `OP_RETURN` ở cuối thân hàm. Giờ, trước đó, nó sẽ phát sinh instruction để push `nil` lên stack. Và với điều đó, ta đã có lời gọi hàm hoạt động! Chúng thậm chí còn nhận tham số! Trông gần như là ta biết mình đang làm gì vậy.

## Return Statements

Nếu bạn muốn một hàm trả về thứ gì đó khác với `nil` ngầm định, bạn cần một câu lệnh `return`. Hãy làm cho nó hoạt động.

^code match-return (1 before, 1 after)

Khi compiler thấy từ khóa `return`, nó sẽ đi tới đây:

^code return-statement

Biểu thức giá trị trả về là tùy chọn, nên parser sẽ tìm token dấu chấm phẩy để xác định xem có giá trị được cung cấp hay không. Nếu không có giá trị trả về, câu lệnh sẽ ngầm trả về `nil`. Ta hiện thực điều đó bằng cách gọi `emitReturn()`, hàm này sẽ phát sinh instruction `OP_NIL`. Ngược lại, ta compile biểu thức giá trị trả về và trả nó bằng instruction `OP_RETURN`.

Đây chính là instruction `OP_RETURN` mà ta đã hiện thực — không cần thêm code runtime mới. Điều này khác khá nhiều so với jlox. Ở đó, ta phải dùng exception để “unwind” stack khi một câu lệnh `return` được execute. Lý do là bạn có thể return từ sâu bên trong các block lồng nhau. Vì jlox duyệt AST một cách đệ quy, điều đó có nghĩa là có hàng loạt lời gọi hàm Java cần phải thoát ra.

Bytecode compiler của chúng ta đã “làm phẳng” tất cả. Ta có đệ quy khi parse, nhưng ở runtime, vòng lặp dispatch bytecode của VM hoàn toàn phẳng. Không có đệ quy nào ở cấp C cả. Vậy nên việc return, ngay cả từ trong các block lồng nhau, cũng đơn giản như return từ cuối thân hàm.

Tuy nhiên, ta vẫn chưa hoàn toàn xong. Câu lệnh `return` mới này mang tới một lỗi compile mới cần quan tâm. Return hữu ích để thoát khỏi hàm, nhưng top-level của một chương trình Lox cũng là code dạng mệnh lệnh. Bạn không nên <span name="worst">return</span> từ đó.

```lox
return "What?!";
```

<aside name="worst">

Cho phép `return` ở top-level cũng không phải ý tưởng tệ nhất. Nó sẽ cho bạn một cách tự nhiên để kết thúc script sớm. Bạn thậm chí có thể dùng một số trả về để biểu thị exit code của tiến trình.

</aside>

Chúng ta đã quy định rằng việc có câu lệnh `return` bên ngoài bất kỳ hàm nào là một lỗi compile, và ta hiện thực điều đó như sau:

^code return-from-script (1 before, 1 after)

Đây là một trong những lý do ta thêm enum FunctionType vào compiler.

## Native Functions

VM của chúng ta đang ngày càng mạnh mẽ hơn. Giờ ta đã có function, lời gọi, tham số, giá trị trả về. Bạn có thể định nghĩa nhiều function khác nhau và cho chúng gọi lẫn nhau theo những cách thú vị. Nhưng, cuối cùng thì, chúng vẫn chưa thực sự _làm_ được gì. Thứ duy nhất mà một chương trình Lox có thể làm cho người dùng thấy, bất kể nó phức tạp đến đâu, là in ra màn hình. Để bổ sung thêm khả năng, ta cần “mở” chúng cho người dùng sử dụng.

Một implementation ngôn ngữ lập trình “vươn ra” và chạm tới thế giới vật chất thông qua **native function**. Nếu bạn muốn viết chương trình kiểm tra thời gian, đọc input từ người dùng, hoặc truy cập hệ thống file, ta cần thêm các native function — có thể gọi từ Lox nhưng được hiện thực bằng C — để cung cấp các khả năng đó.

Về mặt ngôn ngữ, Lox khá đầy đủ — nó có closures, classes, inheritance, và nhiều thứ thú vị khác. Một lý do khiến nó vẫn mang cảm giác như một ngôn ngữ “đồ chơi” là vì nó gần như không có khả năng native nào. Ta có thể biến nó thành một ngôn ngữ “thực thụ” bằng cách thêm một danh sách dài các khả năng này.

Tuy nhiên, việc “cày” qua một đống thao tác hệ điều hành thực ra không mang lại nhiều giá trị học tập. Khi bạn đã thấy cách liên kết một đoạn code C với Lox, bạn sẽ hiểu nguyên lý. Nhưng bạn vẫn cần thấy _một_ ví dụ, và ngay cả một native function duy nhất cũng đòi hỏi ta phải xây dựng toàn bộ cơ chế để kết nối Lox với C. Vậy nên ta sẽ đi qua quá trình đó và làm hết phần khó. Sau đó, khi xong, ta sẽ thêm một native function nhỏ xíu chỉ để chứng minh nó hoạt động.

Lý do ta cần cơ chế mới là vì, từ góc nhìn của implementation, native function khác với Lox function. Khi được gọi, chúng không push một CallFrame, vì không có bytecode nào để frame đó trỏ tới. Chúng không có chunk bytecode. Thay vào đó, chúng tham chiếu tới một đoạn code C native.

Trong clox, ta xử lý điều này bằng cách định nghĩa native function như một loại object hoàn toàn khác.

^code obj-native (1 before, 2 after)

Cách biểu diễn này đơn giản hơn ObjFunction — chỉ gồm một Obj header và một con trỏ tới hàm C hiện thực hành vi native. Native function nhận số lượng đối số và một con trỏ tới đối số đầu tiên trên stack. Nó truy cập các đối số thông qua con trỏ đó. Khi xong, nó trả về giá trị kết quả.

Như thường lệ, một loại object mới sẽ đi kèm một số “phụ kiện”. Để tạo một ObjNative, ta khai báo một hàm giống như constructor.

^code new-native-h (1 before, 1 after)

Ta hiện thực nó như sau:

^code new-native

Constructor nhận một con trỏ hàm C để “bọc” trong ObjNative. Nó thiết lập header của object và lưu hàm đó. Với header, ta cần một loại object mới.

^code obj-type-native (2 before, 2 after)

VM cũng cần biết cách giải phóng một native function object.

^code free-native (1 before, 1 after)

Không có gì nhiều ở đây vì ObjNative không sở hữu thêm vùng nhớ nào khác. Khả năng khác mà mọi object Lox đều hỗ trợ là in ra.

^code print-native (1 before, 1 after)

Để hỗ trợ dynamic typing, ta có một macro để kiểm tra xem một value có phải là native function hay không.

^code is-native (1 before, 1 after)

Nếu macro đó trả về true, macro này sẽ trích xuất con trỏ hàm C từ một Value đại diện cho native function:

^code as-native (1 before, 1 after)

Tất cả những thứ “lỉnh kỉnh” này cho phép VM xử lý native function như bất kỳ object nào khác. Bạn có thể lưu chúng vào biến, truyền đi, thậm chí tổ chức tiệc sinh nhật cho chúng, v.v. Tất nhiên, thao tác mà ta thực sự quan tâm là _gọi_ chúng — dùng một native function làm toán hạng bên trái trong một call expression.

Trong `callValue()`, ta thêm một nhánh xử lý kiểu mới.

^code call-native (2 before, 1 after)

Nếu object được gọi là native function, ta gọi hàm C đó ngay lập tức. Không cần đụng tới CallFrame hay gì cả. Ta chỉ việc chuyển quyền cho C, lấy kết quả, và đặt nó lại vào stack. Điều này khiến native function nhanh nhất có thể.

Với điều này, người dùng có thể gọi native function, nhưng hiện tại chưa có cái nào để gọi. Nếu không có thứ như foreign function interface, người dùng không thể tự định nghĩa native function. Đó là việc của chúng ta với tư cách là người hiện thực VM. Ta sẽ bắt đầu với một helper để định nghĩa một native function mới và “mở” nó cho chương trình Lox.

^code define-native

Hàm này nhận một con trỏ tới hàm C và tên mà nó sẽ được biết tới trong Lox. Ta bọc hàm đó trong một ObjNative rồi lưu nó vào một biến global với tên đã cho.

Có lẽ bạn đang thắc mắc vì sao ta push và pop tên và hàm lên stack. Trông hơi kỳ đúng không? Đây là kiểu việc bạn phải để ý khi <span name="worry">garbage</span> collection xuất hiện. Cả `copyString()` và `newNative()` đều cấp phát bộ nhớ động. Điều đó có nghĩa là khi ta có GC, chúng có thể kích hoạt việc thu gom. Nếu điều đó xảy ra, ta cần đảm bảo bộ thu gom biết rằng ta vẫn chưa xong với tên và ObjFunction để nó không giải phóng chúng. Lưu chúng trên value stack sẽ đảm bảo điều đó.

<aside name="worry">

Đừng lo nếu bạn chưa hiểu hết. Mọi thứ sẽ rõ ràng hơn nhiều khi ta bắt tay vào [hiện thực GC](garbage-collection.html).

</aside>

Nghe có vẻ hơi buồn cười, nhưng sau tất cả đống công việc vừa rồi, chúng ta sẽ chỉ thêm đúng một native function nhỏ.

^code clock-native

Hàm này trả về thời gian đã trôi qua kể từ khi chương trình bắt đầu chạy, tính bằng giây. Nó rất tiện để benchmark các chương trình Lox. Trong Lox, ta sẽ đặt tên nó là `clock()`.

^code define-native-clock (1 before, 1 after)

Để dùng được hàm `clock()` của thư viện chuẩn C, module "vm" cần thêm một include.

^code vm-include-time (1 before, 2 after)

Đó là khá nhiều nội dung để xử lý, nhưng chúng ta đã làm được! Hãy gõ đoạn này vào và thử chạy:

```lox
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 2) + fib(n - 1);
}

var start = clock();
print fib(35);
print clock() - start;
```

Chúng ta có thể viết một hàm Fibonacci đệ quy cực kỳ kém hiệu quả. Thậm chí hay hơn, ta có thể đo được chính xác <span name="faster">_mức độ_</span> kém hiệu quả của nó. Tất nhiên, đây không phải là cách thông minh nhất để tính số Fibonacci. Nhưng nó là một cách tốt để “stress test” khả năng hỗ trợ lời gọi hàm của một implementation ngôn ngữ. Trên máy của tôi, chạy đoạn này trong clox nhanh hơn khoảng năm lần so với jlox. Đó là một cải thiện đáng kể.

<aside name="faster">

Nó chậm hơn một chút so với chương trình Ruby tương đương chạy trên Ruby 2.4.3p205, và nhanh hơn khoảng 3 lần so với khi chạy trên Python 3.7.3. Và chúng ta vẫn còn nhiều tối ưu hóa đơn giản có thể thực hiện trong VM.

</aside>

<div class="challenges">

## Thử thách

1.  Việc đọc và ghi trường `ip` là một trong những thao tác được thực hiện thường xuyên nhất bên trong vòng lặp bytecode. Hiện tại, ta truy cập nó thông qua một con trỏ tới CallFrame hiện tại. Điều này yêu cầu một bước gián tiếp qua con trỏ, có thể buộc CPU bỏ qua cache và truy cập bộ nhớ chính. Đây có thể là một “điểm nghẽn” hiệu năng thực sự.

    Lý tưởng nhất, ta sẽ giữ `ip` trong một thanh ghi (register) của CPU. C không cho phép ta _bắt buộc_ điều đó nếu không dùng inline assembly, nhưng ta có thể cấu trúc code để khuyến khích compiler thực hiện tối ưu hóa này. Nếu ta lưu `ip` trực tiếp trong một biến local của C và đánh dấu nó là `register`, có khả năng cao compiler C sẽ đáp ứng “lời đề nghị lịch sự” này.

    Điều này đồng nghĩa ta cần cẩn thận để nạp và lưu biến local `ip` trở lại đúng CallFrame khi bắt đầu và kết thúc lời gọi hàm. Hãy hiện thực tối ưu hóa này. Viết vài benchmark và xem nó ảnh hưởng thế nào tới hiệu năng. Bạn có nghĩ rằng độ phức tạp code tăng thêm là xứng đáng không?

2.  Lời gọi native function nhanh một phần vì ta không kiểm tra xem lời gọi có truyền đúng số lượng đối số mà hàm mong đợi hay không. Thực ra ta nên làm điều đó, nếu không, một lời gọi sai tới native function mà thiếu đối số có thể khiến hàm đọc vùng nhớ chưa khởi tạo. Hãy thêm kiểm tra arity.

3.  Hiện tại, không có cách nào để một native function báo lỗi runtime. Trong một implementation thực tế, đây là điều cần hỗ trợ vì native function sống trong thế giới C _statically typed_ nhưng lại được gọi từ “vùng đất” Lox _dynamically typed_. Nếu người dùng, chẳng hạn, cố truyền một chuỗi vào `sqrt()`, native function đó cần báo lỗi runtime.

    Hãy mở rộng hệ thống native function để hỗ trợ điều này. Khả năng này ảnh hưởng thế nào tới hiệu năng của lời gọi native?

4.  Thêm một vài native function khác để làm những việc bạn thấy hữu ích. Viết vài chương trình sử dụng chúng. Bạn đã thêm gì? Chúng ảnh hưởng thế nào tới cảm giác về ngôn ngữ và tính thực tiễn của nó?

</div>
