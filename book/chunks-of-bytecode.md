> Nếu bạn thấy mình đang dành gần như toàn bộ thời gian cho lý thuyết, hãy bắt đầu dành chút sự chú ý cho những thứ thực hành; điều đó sẽ cải thiện lý thuyết của bạn. Nếu bạn thấy mình đang dành gần như toàn bộ thời gian cho thực hành, hãy bắt đầu dành chút sự chú ý cho những thứ lý thuyết; điều đó sẽ cải thiện việc thực hành của bạn.
>
> <cite>Donald Knuth</cite>

Chúng ta đã có một bản cài đặt hoàn chỉnh của Lox với jlox, vậy tại sao cuốn sách này vẫn chưa kết thúc? Một phần là vì jlox dựa vào <span name="metal">JVM</span> để làm rất nhiều việc cho chúng ta. Nếu muốn hiểu cách một interpreter hoạt động đến tận “tầng kim loại” (hardware), chúng ta cần tự xây dựng những phần đó.

<aside name="metal">

Tất nhiên, interpreter thứ hai của chúng ta cũng dựa vào thư viện chuẩn C cho những thứ cơ bản như cấp phát bộ nhớ, và trình biên dịch C giúp chúng ta không phải bận tâm đến chi tiết của mã máy bên dưới mà nó đang chạy. Thậm chí, mã máy đó có thể còn được cài đặt dựa trên microcode trong chip. Và C runtime thì dựa vào hệ điều hành để cấp phát các trang bộ nhớ. Nhưng chúng ta phải dừng ở *một điểm nào đó* nếu muốn cuốn sách này còn vừa trên kệ sách của bạn.

</aside>

Một lý do còn cơ bản hơn khiến jlox chưa đủ là vì nó **quá chậm**. Một tree-walk interpreter thì ổn với một số loại ngôn ngữ bậc cao, khai báo. Nhưng với một ngôn ngữ imperative đa dụng — kể cả một ngôn ngữ “scripting” như Lox — thì không ổn chút nào. Hãy xem đoạn script nhỏ này:

```lox
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 1) + fib(n - 2); // [fib]
}

var before = clock();
print fib(40);
var after = clock();
print after - before;
```

<aside name="fib">

Đây là một cách tính số Fibonacci cực kỳ kém hiệu quả. Mục tiêu của chúng ta là xem *interpreter* chạy nhanh đến đâu, chứ không phải xem chúng ta có thể viết một chương trình nhanh đến mức nào. Một chương trình chậm nhưng làm nhiều việc — dù vô nghĩa hay không — lại là một bài test tốt cho mục đích này.

</aside>

Trên laptop của tôi, jlox mất khoảng 72 giây để chạy xong. Một chương trình C tương đương chỉ mất nửa giây. Ngôn ngữ scripting kiểu dynamic typing của chúng ta sẽ không bao giờ nhanh bằng một ngôn ngữ static typing với quản lý bộ nhớ thủ công, nhưng chúng ta cũng không cần chấp nhận việc nó chậm hơn *hơn hai bậc độ lớn*.

Chúng ta có thể chạy jlox qua profiler rồi bắt đầu tinh chỉnh các điểm nóng, nhưng điều đó chỉ giúp được đến một mức nào đó. Mô hình execute — đi bộ qua AST — về cơ bản là một thiết kế sai cho mục tiêu này. Chúng ta không thể “vi chỉnh” nó để đạt hiệu năng mong muốn, cũng như bạn không thể đánh bóng một chiếc AMC Gremlin thành một chiếc SR-71 Blackbird.

Chúng ta cần nghĩ lại mô hình cốt lõi. Chương này sẽ giới thiệu mô hình đó — bytecode — và bắt đầu xây dựng interpreter mới của chúng ta, clox.

## Bytecode?

Trong kỹ thuật, hiếm khi có lựa chọn nào mà không phải đánh đổi. Để hiểu rõ tại sao chúng ta chọn bytecode, hãy so sánh nó với một vài phương án khác.

### Tại sao không đi bộ qua AST?

Interpreter hiện tại của chúng ta có vài điểm mạnh:

*   Trước hết, chúng ta đã viết xong nó rồi. Hoàn thành. Và lý do chính là vì kiểu interpreter này *rất dễ cài đặt*. Biểu diễn runtime của code ánh xạ trực tiếp với cú pháp. Từ parser sang cấu trúc dữ liệu cần cho runtime gần như không tốn công sức.

*   Nó *portable*. Interpreter hiện tại được viết bằng Java và chạy trên bất kỳ nền tảng nào Java hỗ trợ. Chúng ta hoàn toàn có thể viết một bản mới bằng C với cùng cách tiếp cận, rồi biên dịch và chạy ngôn ngữ của mình trên hầu như mọi nền tảng.

Đó là những lợi thế thật sự. Nhưng, ngược lại, nó *không tiết kiệm bộ nhớ*. Mỗi mẩu cú pháp trở thành một node AST. Một biểu thức Lox nhỏ như `1 + 2` lại biến thành một đống object với hàng loạt con trỏ liên kết, trông như thế này:

<span name="header"></span>

<aside name="header">

Phần “(header)” là thông tin quản lý mà Java virtual machine dùng để hỗ trợ quản lý bộ nhớ và lưu kiểu của object. Chúng cũng chiếm dung lượng!

</aside>

<img src="image/chunks-of-bytecode/ast.png" alt="The tree of Java objects created to represent '1 + 2'." />

Mỗi con trỏ như vậy lại thêm 32 hoặc 64 bit overhead vào object. Tệ hơn, việc rải dữ liệu khắp heap trong một mạng lưới object lỏng lẻo gây ra vấn đề nghiêm trọng cho <span name="locality">*spatial locality*</span>.

<aside name="locality">

Tôi đã viết [một chương riêng](http://gameprogrammingpatterns.com/data-locality.html) về chính vấn đề này trong cuốn sách đầu tiên của mình, *Game Programming Patterns*, nếu bạn muốn tìm hiểu sâu hơn.

</aside>

CPU hiện đại xử lý dữ liệu nhanh hơn nhiều so với tốc độ lấy dữ liệu từ RAM. Để bù lại, chip có nhiều tầng cache. Nếu một vùng nhớ đã có sẵn trong cache, nó có thể được nạp nhanh hơn rất nhiều — nhanh hơn tới 100 *lần*.

Dữ liệu vào cache bằng cách nào? Bộ xử lý sẽ “đoán” và nạp sẵn cho bạn. Nguyên tắc khá đơn giản: mỗi khi CPU đọc một chút dữ liệu từ RAM, nó sẽ lấy luôn một cụm byte liền kề và đưa vào cache.

Nếu chương trình của chúng ta tiếp theo cần dữ liệu nằm trong cùng cache line đó, CPU sẽ chạy mượt như một dây chuyền sản xuất trơn tru. Chúng ta *rất* muốn tận dụng điều này. Để dùng cache hiệu quả, cách chúng ta biểu diễn code trong bộ nhớ nên đặc và có thứ tự giống như khi nó được đọc.

Giờ hãy nhìn lại cái cây ở trên. Những sub-object đó có thể nằm <span name="anywhere">*bất cứ đâu*</span>. Mỗi bước mà tree-walker đi theo một tham chiếu đến node con có thể vượt ra ngoài phạm vi cache và buộc CPU phải chờ cho đến khi một cụm dữ liệu mới được nạp từ RAM. Chỉ riêng *overhead* của các node trong cây với tất cả các trường con trỏ và header object đã đủ để đẩy chúng ra xa nhau và ra khỏi cache.

<aside name="anywhere">

Ngay cả khi các object tình flag được cấp phát liên tiếp trong bộ nhớ khi parser vừa tạo ra chúng, thì sau vài vòng garbage collection — vốn có thể di chuyển object trong bộ nhớ — cũng chẳng thể biết chúng sẽ nằm ở đâu.

</aside>

AST walker của chúng ta còn có overhead khác như interface dispatch và Visitor pattern, nhưng chỉ riêng vấn đề locality cũng đã đủ để biện minh cho một cách biểu diễn code tốt hơn.


### Tại sao không biên dịch thẳng sang native code?

Nếu bạn muốn chạy *thật* nhanh, bạn sẽ muốn loại bỏ tất cả những tầng gián tiếp đó. Xuống tận “tầng kim loại”. Machine code. Nghe thôi cũng đã thấy nhanh rồi. *Machine code.*

Biên dịch trực tiếp sang tập lệnh gốc mà chip hỗ trợ là cách mà các ngôn ngữ nhanh nhất vẫn làm. Nhắm tới native code đã là lựa chọn hiệu quả nhất từ những ngày đầu, khi các kỹ sư thực sự <span name="hand">viết tay</span> chương trình bằng machine code.

<aside name="hand">

Vâng, họ thực sự viết machine code bằng tay. Trên thẻ đục lỗ. Và có lẽ, họ đục lỗ *bằng nắm đấm*.

</aside>

Nếu bạn chưa từng viết machine code, hoặc “người anh em” dễ đọc hơn một chút là assembly code, thì tôi sẽ giới thiệu nhẹ nhàng nhất có thể. Native code là một chuỗi dày đặc các thao tác, được mã hóa trực tiếp ở dạng nhị phân. Mỗi lệnh dài từ một đến vài byte, và gần như ở mức thấp đến mức… tê liệt não. “Chuyển một giá trị từ địa chỉ này sang thanh ghi kia.” “Cộng hai số nguyên trong hai thanh ghi này.” Đại loại như vậy.

CPU sẽ chạy qua các lệnh đó, giải mã và execute từng lệnh theo thứ tự. Không có cấu trúc cây như AST của chúng ta, và luồng điều khiển được xử lý bằng cách nhảy từ điểm này sang điểm khác trong code. Không gián tiếp, không overhead, không nhảy lung tung hay lần theo con trỏ.

Nhanh như chớp, nhưng hiệu năng đó phải trả giá. Trước hết, biên dịch sang native code không hề dễ. Hầu hết các chip phổ biến ngày nay đều có kiến trúc phức tạp kiểu Byzantine với hàng đống lệnh tích tụ qua nhiều thập kỷ. Chúng đòi hỏi kỹ thuật phân bổ thanh ghi tinh vi, pipelining, và sắp xếp lệnh.

Và tất nhiên, bạn đã vứt bỏ <span name="back">tính portable</span>. Dành vài năm để thành thạo một kiến trúc nào đó thì bạn cũng chỉ mới chạy được trên *một* trong số nhiều tập lệnh phổ biến. Muốn ngôn ngữ của mình chạy trên tất cả, bạn phải học tất cả các tập lệnh đó và viết một back end riêng cho từng cái.

<aside name="back">

Tình hình cũng không hoàn toàn bi đát. Một compiler được thiết kế tốt cho phép bạn chia sẻ front end và hầu hết các bước tối ưu hóa ở tầng giữa giữa các kiến trúc khác nhau. Chủ yếu chỉ có phần sinh code và một số chi tiết về chọn lệnh là bạn phải viết lại mỗi lần.

Dự án [LLVM](https://llvm.org/) cung cấp sẵn cho bạn một phần việc này. Nếu compiler của bạn xuất ra ngôn ngữ trung gian đặc biệt của LLVM, thì LLVM sẽ biên dịch nó sang native code cho hàng loạt kiến trúc.

</aside>

### Bytecode là gì?

Hãy ghi nhớ hai điểm này. Ở một đầu, tree-walk interpreter thì đơn giản, portable, nhưng chậm. Ở đầu kia, native code thì phức tạp, phụ thuộc nền tảng nhưng nhanh. Bytecode nằm ở giữa. Nó giữ được tính portable của tree-walker — trong cuốn sách này, chúng ta sẽ không phải đụng tay vào assembly code. Nó hy sinh *một chút* sự đơn giản để đổi lấy hiệu năng cao hơn, dù vẫn không nhanh bằng native code hoàn toàn.

Về cấu trúc, bytecode giống machine code. Nó là một chuỗi tuyến tính, dày đặc các lệnh nhị phân. Điều này giúp giảm overhead và tận dụng cache tốt hơn. Tuy nhiên, nó là một tập lệnh đơn giản hơn nhiều, ở mức trừu tượng cao hơn bất kỳ con chip thực nào. (Trong nhiều định dạng bytecode, mỗi lệnh chỉ dài đúng một byte, vì thế mới gọi là “bytecode”.)

Hãy tưởng tượng bạn đang viết một native compiler từ một ngôn ngữ nguồn nào đó và được toàn quyền thiết kế kiến trúc dễ nhắm tới nhất. Bytecode gần giống như vậy. Nó là một tập lệnh lý tưởng hóa, giúp cuộc sống của người viết compiler dễ dàng hơn.

Vấn đề của một kiến trúc “ảo tưởng” là… nó không tồn tại. Chúng ta giải quyết điều đó bằng cách viết một *emulator* — một con chip giả lập bằng phần mềm, đọc và execute bytecode từng lệnh một. Hay nói cách khác là một *virtual machine (VM)*.

Lớp giả lập này thêm <span name="p-code">overhead</span>, đây là lý do chính khiến bytecode chậm hơn native code. Nhưng đổi lại, nó cho chúng ta tính portable. Viết VM bằng một ngôn ngữ như C — vốn đã được hỗ trợ trên mọi máy mà ta quan tâm — và ta có thể chạy emulator đó trên bất kỳ phần cứng nào mình muốn.

<aside name="p-code">

Một trong những định dạng bytecode đầu tiên là [p-code](https://en.wikipedia.org/wiki/P-code_machine), được phát triển cho ngôn ngữ Pascal của Niklaus Wirth. Bạn có thể nghĩ rằng một PDP-11 chạy ở 15MHz thì không thể chịu nổi overhead của việc giả lập một virtual machine. Nhưng thời đó, máy tính đang trong thời kỳ “bùng nổ Cambrian” và kiến trúc mới xuất hiện liên tục. Bắt kịp chip mới quan trọng hơn là vắt kiệt hiệu năng từ từng con chip. Đó là lý do chữ “p” trong p-code không phải là “Pascal” mà là “portable”.

</aside>

Đây chính là con đường chúng ta sẽ đi với interpreter mới, clox. Chúng ta sẽ theo bước các bản cài đặt chính của Python, Ruby, Lua, OCaml, Erlang, và nhiều ngôn ngữ khác. Ở nhiều khía cạnh, thiết kế VM của chúng ta sẽ song song với cấu trúc của interpreter trước:

<img src="image/chunks-of-bytecode/phases.png" alt="Phases of the two
implementations. jlox is Parser to Syntax Trees to Interpreter. clox is Compiler
to Bytecode to Virtual Machine." />

Tất nhiên, chúng ta sẽ không cài đặt các giai đoạn này theo đúng thứ tự. Giống như interpreter trước, chúng ta sẽ “nhảy qua nhảy lại”, xây dựng từng tính năng ngôn ngữ một. Trong chương này, chúng ta sẽ dựng bộ khung của ứng dụng và tạo các cấu trúc dữ liệu cần thiết để lưu trữ và biểu diễn một chunk bytecode.

## Bắt đầu thôi

Còn bắt đầu từ đâu nữa, nếu không phải là `main()`? <span name="ready">Mở</span> trình soạn thảo quen thuộc của bạn và bắt đầu gõ.

<aside name="ready">

Đây là lúc thích hợp để vươn vai, bẻ khớp ngón tay. Thêm chút nhạc nền kiểu montage cũng không tệ.

</aside>

^code main-c

Từ hạt giống nhỏ bé này, chúng ta sẽ phát triển toàn bộ VM. Vì C cung cấp cho chúng ta quá ít, nên trước tiên ta cần “bón đất” một chút. Một phần việc đó nằm trong file header này:

^code common-h

Có một vài kiểu dữ liệu và hằng số mà chúng ta sẽ dùng xuyên suốt interpreter, và đây là nơi tiện lợi để đặt chúng. Hiện tại, đó là `NULL` quen thuộc, `size_t`, kiểu Boolean `bool` của C99, và các kiểu số nguyên có kích thước rõ ràng — `uint8_t` và các “người bạn” của nó.


## Các khối lệnh (Chunks of Instructions)

Tiếp theo, chúng ta cần một module để định nghĩa cách biểu diễn code. Tôi đã dùng từ “chunk” để chỉ các chuỗi bytecode, vậy nên hãy chính thức dùng nó làm tên cho module này.

^code chunk-h

Trong định dạng bytecode của chúng ta, mỗi instruction có một byte **operation code** (thường được viết tắt là **opcode**). Con số này quyết định loại instruction mà chúng ta đang xử lý — cộng, trừ, tra cứu biến, v.v. Chúng ta sẽ định nghĩa chúng ở đây:

^code op-enum (1 before, 2 after)

Hiện tại, ta bắt đầu với một instruction duy nhất, `OP_RETURN`. Khi VM của chúng ta đầy đủ tính năng, instruction này sẽ có nghĩa là “trả về từ hàm hiện tại”. Tôi thừa nhận là nó chưa hữu ích lắm, nhưng ta phải bắt đầu từ đâu đó, và đây là một instruction đặc biệt đơn giản, vì những lý do mà ta sẽ nói đến sau.

### Mảng động chứa các instruction

Bytecode là một chuỗi các instruction. Sau này, chúng ta sẽ lưu thêm một số dữ liệu khác cùng với các instruction, nên hãy tạo luôn một struct để chứa tất cả.

^code chunk-struct (1 before, 2 after)

Hiện tại, nó chỉ đơn giản là một lớp bọc quanh một mảng byte. Vì chúng ta không biết trước kích thước mảng cần bao nhiêu trước khi bắt đầu compile một chunk, nên nó phải là mảng động. Mảng động là một trong những cấu trúc dữ liệu tôi yêu thích nhất. Nghe có vẻ giống như việc nói “vanilla là vị kem yêu thích của tôi” <span name="flavor">vậy</span>, nhưng hãy nghe tôi giải thích. Mảng động mang lại:

<aside name="flavor">

Thực ra, butter pecan mới là vị tôi thích nhất.

</aside>

* Lưu trữ dày đặc, thân thiện với cache  
* Truy cập phần tử theo chỉ số trong thời gian hằng số  
* Thêm phần tử vào cuối mảng trong thời gian hằng số  

Chính những đặc điểm này là lý do chúng ta dùng mảng động liên tục trong jlox, dưới dạng lớp ArrayList của Java. Giờ đây khi ở trong C, chúng ta sẽ tự viết lấy. Nếu bạn đã quên cách mảng động hoạt động, ý tưởng khá đơn giản: ngoài mảng dữ liệu, ta giữ thêm hai con số — số phần tử đã được cấp phát (“capacity”) và số phần tử thực sự đang dùng (“count”).

^code count-and-capacity (1 before, 2 after)

Khi thêm một phần tử, nếu `count` nhỏ hơn `capacity`, tức là mảng vẫn còn chỗ trống. Ta chỉ cần lưu phần tử mới vào đó và tăng `count`.

<img src="image/chunks-of-bytecode/insert.png" alt="Storing an element in an array that has enough capacity." />

Nếu không còn chỗ trống, quá trình sẽ phức tạp hơn một chút.

<img src="image/chunks-of-bytecode/grow.png" alt="Growing the dynamic array before storing an element." class="wide" />

1.  <span name="amortized">Cấp phát</span> một mảng mới với dung lượng lớn hơn.  
2.  Sao chép các phần tử hiện có từ mảng cũ sang mảng mới.  
3.  Lưu `capacity` mới.  
4.  Giải phóng mảng cũ.  
5.  Cập nhật `code` để trỏ tới mảng mới.  
6.  Lưu phần tử mới vào mảng mới khi đã có chỗ.  
7.  Cập nhật `count`.  

<aside name="amortized">

Việc sao chép các phần tử khi mở rộng mảng khiến nó có vẻ như thao tác thêm phần tử là *O(n)*, chứ không phải *O(1)* như tôi vừa nói. Tuy nhiên, bạn chỉ cần sao chép ở *một số* lần thêm. Phần lớn thời gian, mảng đã có sẵn chỗ trống nên không cần sao chép.

Để hiểu điều này, chúng ta cần [**amortized analysis**](https://en.wikipedia.org/wiki/Amortized_analysis). Phân tích này cho thấy miễn là ta mở rộng mảng theo bội số của kích thước hiện tại, khi tính trung bình chi phí của *một chuỗi* thao tác thêm, mỗi lần thêm vẫn là *O(1)*.

</aside>

Chúng ta đã có struct, giờ hãy viết các hàm để làm việc với nó. C không có constructor, nên ta khai báo một hàm để khởi tạo một chunk mới.

^code init-chunk-h (1 before, 2 after)

Và cài đặt nó như sau:

^code chunk-c

Mảng động bắt đầu hoàn toàn rỗng. Ta thậm chí chưa cấp phát mảng thô. Để thêm một byte vào cuối chunk, ta dùng một hàm mới.

^code write-chunk-h (1 before, 2 after)

Đây là nơi công việc thú vị diễn ra.

^code write-chunk

Việc đầu tiên là kiểm tra xem mảng hiện tại đã đủ chỗ cho byte mới chưa. Nếu chưa, ta cần mở rộng mảng để có chỗ. (Trường hợp này cũng xảy ra ngay lần ghi đầu tiên khi mảng là `NULL` và `capacity` bằng 0.)

Để mở rộng mảng, trước tiên ta tính toán `capacity` mới và mở rộng mảng tới kích thước đó. Cả hai thao tác cấp phát bộ nhớ cấp thấp này được định nghĩa trong một module mới.

^code chunk-c-include-memory (1 before, 2 after)

Vậy là đủ để bắt đầu.

^code memory-h

Macro này tính toán `capacity` mới dựa trên `capacity` hiện tại. Để đạt hiệu năng mong muốn, điều quan trọng là nó phải *tăng theo tỷ lệ* so với kích thước cũ. Chúng ta tăng gấp đôi, đây là cách khá phổ biến. 1.5× cũng là một lựa chọn thường gặp.

Ta cũng xử lý trường hợp `capacity` hiện tại bằng 0. Khi đó, ta nhảy thẳng lên 8 phần tử thay vì bắt đầu từ 1. Điều này <span name="profile">tránh</span> việc cấp phát lại quá nhiều khi mảng còn rất nhỏ, đổi lại là lãng phí vài byte cho các chunk rất nhỏ.

<aside name="profile">

Tôi chọn số 8 khá tùy ý cho cuốn sách này. Hầu hết các cài đặt mảng động đều có một ngưỡng tối thiểu như vậy. Cách đúng đắn để chọn giá trị là đo đạc trên dữ liệu thực tế và tìm hằng số cho hiệu năng tốt nhất giữa số lần mở rộng và dung lượng lãng phí.

</aside>

Khi đã biết `capacity` mong muốn, ta tạo mới hoặc mở rộng mảng tới kích thước đó bằng `GROW_ARRAY()`.

^code grow-array (2 before, 2 after)

Macro này giúp lời gọi hàm `reallocate()` trông gọn gàng hơn. Nó lo việc lấy kích thước phần tử của mảng và ép kiểu con trỏ `void*` trả về thành đúng kiểu con trỏ cần dùng.

Hàm `reallocate()` này là hàm duy nhất chúng ta sẽ dùng cho mọi thao tác quản lý bộ nhớ động trong clox — cấp phát, giải phóng, và thay đổi kích thước vùng nhớ đã cấp phát. Gom tất cả các thao tác này qua một hàm duy nhất sẽ rất quan trọng sau này khi ta thêm garbage collector để theo dõi lượng bộ nhớ đang dùng.

Hai tham số kích thước truyền vào `reallocate()` quyết định thao tác sẽ thực hiện:

<table>
  <thead>
    <tr>
      <td>oldSize</td>
      <td>newSize</td>
      <td>Thao tác</td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>Khác&nbsp;0</td>
      <td>Cấp phát block mới.</td>
    </tr>
    <tr>
      <td>Khác&nbsp;0</td>
      <td>0</td>
      <td>Giải phóng vùng nhớ.</td>
    </tr>
    <tr>
      <td>Khác&nbsp;0</td>
      <td>Nhỏ&nbsp;hơn&nbsp;<code>oldSize</code></td>
      <td>Thu nhỏ vùng nhớ hiện có.</td>
    </tr>
    <tr>
      <td>Khác&nbsp;0</td>
      <td>Lớn&nbsp;hơn&nbsp;<code>oldSize</code></td>
      <td>Mở rộng vùng nhớ hiện có.</td>
    </tr>
  </tbody>
</table>

Nghe có vẻ như phải xử lý khá nhiều trường hợp, nhưng đây là phần cài đặt:

^code memory-c

Khi `newSize` bằng 0, chúng ta tự xử lý việc giải phóng bộ nhớ bằng cách gọi `free()`. Ngược lại, ta dựa vào hàm `realloc()` của thư viện chuẩn C. Hàm này tiện lợi ở chỗ nó hỗ trợ luôn ba trường hợp còn lại trong “chính sách” của chúng ta. Khi `oldSize` bằng 0, `realloc()` tương đương với việc gọi `malloc()`.

Các trường hợp thú vị là khi cả `oldSize` và `newSize` đều khác 0. Khi đó, `realloc()` sẽ thay đổi kích thước của block bộ nhớ đã được cấp phát trước đó. Nếu kích thước mới nhỏ hơn block hiện tại, nó chỉ đơn giản <span name="shrink">cập nhật</span> kích thước của block và trả về cùng con trỏ bạn đã đưa vào. Nếu kích thước mới lớn hơn, nó sẽ cố gắng mở rộng block bộ nhớ hiện có.

Điều này chỉ thực hiện được nếu vùng nhớ ngay sau block đó chưa được sử dụng. Nếu không còn chỗ để mở rộng, `realloc()` sẽ cấp phát một block bộ nhớ *mới* với kích thước mong muốn, sao chép dữ liệu cũ sang, giải phóng block cũ, rồi trả về con trỏ tới block mới. Hãy nhớ rằng, đây chính xác là hành vi mà chúng ta muốn cho mảng động của mình.

Vì máy tính là những khối vật chất hữu hạn chứ không phải các mô hình toán học hoàn hảo như lý thuyết khoa học máy tính thường giả định, việc cấp phát có thể thất bại nếu không đủ bộ nhớ, và `realloc()` sẽ trả về `NULL`. Chúng ta cần xử lý tình huống này.

^code out-of-memory (1 before, 1 after)

Thực ra, nếu VM không thể lấy được bộ nhớ cần thiết thì cũng chẳng còn gì *hữu ích* để làm, nhưng ít nhất ta sẽ phát hiện ra và dừng chương trình ngay lập tức, thay vì trả về con trỏ `NULL` và để mọi thứ “trật đường ray” về sau.

<aside name="shrink">

Vì tất cả những gì ta truyền vào chỉ là một con trỏ trần trụi tới byte đầu tiên của vùng nhớ, vậy “cập nhật” kích thước block nghĩa là gì? Ẩn bên dưới, bộ cấp phát bộ nhớ duy trì thêm thông tin quản lý cho mỗi block bộ nhớ trên heap, bao gồm cả kích thước của nó.

Khi có một con trỏ tới vùng nhớ đã được cấp phát, bộ cấp phát có thể tìm thấy thông tin quản lý này, vốn cần thiết để giải phóng bộ nhớ một cách an toàn. Chính metadata về kích thước này là thứ mà `realloc()` cập nhật.

Nhiều cài đặt của `malloc()` lưu kích thước đã cấp phát ngay *trước* địa chỉ trả về.

</aside>

OK, giờ chúng ta có thể tạo các chunk mới và ghi instruction vào đó. Xong chưa? Chưa đâu! Chúng ta đang ở trong C, nhớ chứ, nên phải tự quản lý bộ nhớ, như “thời xưa”, và điều đó có nghĩa là cũng phải *giải phóng* nó.

^code free-chunk-h (1 before, 1 after)

Phần cài đặt như sau:

^code free-chunk

Chúng ta giải phóng toàn bộ bộ nhớ, rồi gọi `initChunk()` để đặt lại các trường về trạng thái rỗng, rõ ràng. Để giải phóng bộ nhớ, ta thêm một macro nữa.

^code free-array (3 before, 2 after)

Giống như `GROW_ARRAY()`, macro này là một lớp bọc quanh lời gọi `reallocate()`. Macro này giải phóng bộ nhớ bằng cách truyền vào giá trị 0 cho kích thước mới. Tôi biết, đây là khá nhiều chi tiết cấp thấp nhàm chán. Nhưng đừng lo, chúng ta sẽ dùng chúng rất nhiều ở các chương sau và sẽ được lập trình ở mức cao hơn. Trước khi làm được điều đó, ta phải tự xây nền móng đã.

## Giải mã (Disassembling) các Chunk

Giờ chúng ta đã có một module nhỏ để tạo các chunk bytecode. Hãy thử nó bằng cách tự tay tạo một chunk mẫu.

^code main-chunk (1 before, 1 after)

Đừng quên include.

^code main-include-chunk (1 before, 2 after)

Chạy thử và xem kết quả. Nó có hoạt động không? Ờ… ai mà biết được? Tất cả những gì ta làm là đẩy vài byte vào bộ nhớ. Chúng ta chưa có cách nào thân thiện với con người để xem bên trong chunk vừa tạo.

Để khắc phục, chúng ta sẽ tạo một **disassembler**. Một **assembler** là chương trình “cổ điển” nhận một file chứa các tên mnemonic dễ đọc của con người cho các lệnh CPU như “ADD” và “MULT” rồi dịch chúng sang mã máy nhị phân tương ứng. Một *dis*assembler thì đi theo hướng ngược lại — nhận một khối mã máy và xuất ra danh sách các instruction ở dạng văn bản.

Chúng ta sẽ cài đặt một thứ <span name="printer">tương tự</span>. Cho một chunk, nó sẽ in ra tất cả các instruction trong đó. Người *dùng* Lox sẽ không dùng công cụ này, nhưng chúng ta — những người *duy trì* Lox — sẽ thấy nó cực kỳ hữu ích vì nó cho ta một “cửa sổ” nhìn vào cách interpreter biểu diễn code bên trong.

<aside name="printer">

Trong jlox, công cụ tương tự của chúng ta là [AstPrinter class](representing-code.html#a-not-very-pretty-printer).

</aside>

Trong `main()`, sau khi tạo chunk, ta truyền nó vào disassembler.

^code main-disassemble-chunk (2 before, 1 after)

Và lại tạo thêm <span name="module">một module nữa</span>.

<aside name="module">

Tôi hứa là ở các chương sau sẽ không tạo nhiều file mới như thế này nữa.

</aside>

^code main-include-debug (1 before, 2 after)

Đây là phần header:

^code debug-h

Trong `main()`, chúng ta gọi `disassembleChunk()` để giải mã (disassemble) toàn bộ các instruction trong cả chunk. Hàm này được cài đặt dựa trên một hàm khác, chỉ giải mã một instruction duy nhất. Nó xuất hiện ở phần header vì sau này chúng ta sẽ gọi nó từ VM trong các chương tiếp theo.

Dưới đây là phần khởi đầu của file cài đặt:

^code debug-c

Để giải mã một chunk, chúng ta in ra một tiêu đề nhỏ (để biết *chunk nào* đang được xem), rồi duyệt qua bytecode, giải mã từng instruction. Cách chúng ta lặp qua code hơi khác thường: thay vì tăng `offset` ngay trong vòng lặp, ta để `disassembleInstruction()` làm việc đó. Khi gọi hàm này, sau khi giải mã instruction tại offset cho trước, nó sẽ trả về offset của instruction *tiếp theo*. Lý do là vì, như ta sẽ thấy sau, các instruction có thể có kích thước khác nhau.

Trung tâm của module "debug" là hàm này:

^code disassemble-instruction

Đầu tiên, nó in ra byte offset của instruction — cho ta biết instruction này nằm ở đâu trong chunk. Đây sẽ là một “cột mốc” hữu ích khi ta bắt đầu xử lý control flow và nhảy qua lại trong bytecode.

Tiếp theo, nó đọc một byte từ bytecode tại offset cho trước. Đây chính là opcode. Chúng ta <span name="switch">switch</span> trên giá trị đó. Với mỗi loại instruction, ta gọi một hàm tiện ích nhỏ để hiển thị nó. Trong trường hợp byte đó hoàn toàn không giống một instruction hợp lệ — tức là có bug trong compiler — ta cũng sẽ in ra. Với instruction duy nhất hiện tại là `OP_RETURN`, hàm hiển thị sẽ là:

<aside name="switch">

Hiện giờ chúng ta chỉ có một instruction, nhưng phần switch này sẽ được mở rộng dần trong suốt cuốn sách.

</aside>

^code simple-instruction

Một return instruction thì chẳng có gì nhiều, nên nó chỉ in ra tên opcode, rồi trả về offset của byte tiếp theo sau instruction này. Các instruction khác sẽ phức tạp hơn.

Nếu chạy interpreter “sơ sinh” của chúng ta bây giờ, nó sẽ in ra:

```text
== test chunk ==
0000 OP_RETURN
```

Hoạt động rồi! Đây giống như “Hello, world!” của cách biểu diễn code. Chúng ta có thể tạo một chunk, ghi một instruction vào đó, rồi đọc lại instruction đó. Việc mã hóa và giải mã bytecode nhị phân của chúng ta đang chạy tốt.

## Constants

Giờ khi đã có một cấu trúc chunk cơ bản hoạt động, hãy bắt đầu làm nó hữu ích hơn. Chúng ta có thể lưu *code* trong chunk, nhưng còn *data* thì sao? Nhiều giá trị mà interpreter xử lý được tạo ra tại runtime như kết quả của các phép toán.

```lox
1 + 2;
```

Giá trị 3 không hề xuất hiện trực tiếp trong code. Tuy nhiên, các literal `1` và `2` thì có. Để compile câu lệnh này thành bytecode, chúng ta cần một loại instruction có nghĩa là “tạo ra một hằng số” và các literal này cần được lưu ở đâu đó trong chunk. Trong jlox, node AST `Expr.Literal` giữ giá trị này. Giờ khi không còn syntax tree, ta cần một giải pháp khác.

### Biểu diễn giá trị

Chúng ta sẽ chưa *chạy* code trong chương này, nhưng vì constants liên quan đến cả phần tĩnh và động của interpreter, chúng buộc ta phải bắt đầu nghĩ về cách VM sẽ biểu diễn giá trị.

Trước mắt, ta sẽ bắt đầu đơn giản nhất có thể — chỉ hỗ trợ số thực dấu chấm động double-precision. Dĩ nhiên, điều này sẽ được mở rộng sau, nên ta sẽ tạo một module mới để có chỗ phát triển.

^code value-h

`typedef` này trừu tượng hóa cách giá trị Lox được biểu diễn cụ thể trong C. Nhờ vậy, ta có thể thay đổi cách biểu diễn mà không phải sửa lại code hiện có đang truyền các giá trị này.

Quay lại câu hỏi: lưu constants ở đâu trong chunk? Với các giá trị nhỏ, kích thước cố định như số nguyên, nhiều tập lệnh lưu giá trị trực tiếp trong luồng code ngay sau opcode. Đây được gọi là **immediate instructions** vì các bit của giá trị nằm ngay sau opcode.

Cách này không phù hợp với các constants lớn hoặc có kích thước thay đổi như chuỗi. Trong compiler native sang machine code, các constants lớn hơn sẽ được lưu ở một vùng “constant data” riêng trong file execute nhị phân. Khi đó, instruction để load một constant sẽ chứa địa chỉ hoặc offset trỏ tới vị trí giá trị đó trong vùng này.

Hầu hết các virtual machine cũng làm tương tự. Ví dụ, Java Virtual Machine [gắn một **constant pool**](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.4) với mỗi class đã compile. Điều này nghe cũng hợp lý cho clox. Mỗi chunk sẽ mang theo một danh sách các giá trị xuất hiện dưới dạng literal trong chương trình. Để <span name="immediate">đơn giản</span> hơn, ta sẽ cho *tất cả* constants vào đây, kể cả số nguyên đơn giản.

<aside name="immediate">

Ngoài việc cần hai loại instruction constant — một cho immediate values và một cho constants trong bảng constant — immediate còn buộc ta phải quan tâm đến alignment, padding, và endianness. Một số kiến trúc không “vui vẻ” nếu bạn cố nhét một số nguyên 4 byte vào một địa chỉ lẻ.

</aside>

### Mảng giá trị (Value arrays)

Constant pool là một mảng các giá trị. Instruction để load một constant sẽ tra cứu giá trị đó theo chỉ số trong mảng. Giống như mảng <span name="generic">bytecode</span> của chúng ta, compiler không biết trước mảng này cần lớn đến mức nào. Vậy nên, một lần nữa, chúng ta cần một mảng động. Vì C không có cấu trúc dữ liệu generic, chúng ta sẽ viết một cấu trúc mảng động khác, lần này dành cho Value.

<aside name="generic">

Việc phải định nghĩa một struct mới và các hàm thao tác mỗi khi cần một mảng động cho kiểu dữ liệu khác nhau khá là phiền. Chúng ta có thể ghép vài macro của preprocessor để giả lập generic, nhưng như vậy là quá mức cần thiết cho clox. Chúng ta sẽ không cần nhiều mảng động kiểu này đâu.

</aside>

^code value-array (1 before, 2 after)

Giống như mảng bytecode trong Chunk, struct này bọc một con trỏ tới mảng cùng với dung lượng đã cấp phát và số phần tử đang sử dụng. Chúng ta cũng cần ba hàm tương tự để làm việc với mảng giá trị.

^code array-fns-h (1 before, 2 after)

Phần cài đặt có thể sẽ khiến bạn thấy “quen quen”. Đầu tiên, để tạo một mảng mới:

^code value-c

Khi đã có một mảng được khởi tạo, ta có thể bắt đầu <span name="add">thêm</span> giá trị vào nó.

<aside name="add">

May mắn là chúng ta không cần các thao tác khác như chèn hoặc xóa.

</aside>

^code write-value-array

Các macro quản lý bộ nhớ mà ta đã viết trước đó cho phép tái sử dụng một phần logic từ mảng code, nên việc này cũng không quá tệ. Cuối cùng, để giải phóng toàn bộ bộ nhớ mà mảng sử dụng:

^code free-value-array

Giờ khi đã có mảng giá trị có thể mở rộng, chúng ta có thể thêm một mảng như vậy vào Chunk để lưu constants của chunk.

^code chunk-constants (1 before, 1 after)

Đừng quên include.

^code chunk-h-include-value (1 before, 2 after)

Ôi C, và câu chuyện modularity thời “đồ đá” của nó. Chúng ta đang ở đâu rồi nhỉ? À đúng, khi khởi tạo một chunk mới, ta cũng khởi tạo luôn danh sách constants của nó.

^code chunk-init-constant-array (1 before, 1 after)

Tương tự, ta giải phóng constants khi giải phóng chunk.

^code chunk-free-constants (1 before, 1 after)

Tiếp theo, chúng ta định nghĩa một hàm tiện lợi để thêm một constant mới vào chunk. Compiler (chưa được viết) của chúng ta hoàn toàn có thể ghi trực tiếp vào mảng constant bên trong Chunk — vì C đâu có khái niệm private field — nhưng sẽ gọn gàng hơn nếu có một hàm rõ ràng cho việc này.

^code add-constant-h (1 before, 2 after)

Rồi ta cài đặt nó.

^code add-constant

Sau khi thêm constant, ta trả về chỉ số nơi constant được thêm vào để có thể tìm lại constant đó sau này.

### Constant instructions

Chúng ta có thể *lưu* constants trong chunk, nhưng cũng cần *execute* chúng. Trong một đoạn code như:

```lox
print 1;
print 2;
```

Chunk đã compile không chỉ cần chứa giá trị 1 và 2, mà còn phải biết *khi nào* tạo ra chúng để in ra đúng thứ tự. Vì vậy, chúng ta cần một instruction tạo ra một constant cụ thể.

^code op-constant (1 before, 1 after)

Khi VM execute một constant instruction, nó sẽ <span name="load">"load"</span> constant đó để sử dụng. Instruction mới này phức tạp hơn một chút so với `OP_RETURN`. Trong ví dụ trên, chúng ta load hai constant khác nhau. Một opcode trần trụi là không đủ để biết *constant nào* cần load.

<aside name="load">

Tôi đang cố tình nói mơ hồ về việc “load” hay “tạo ra” một constant vì chúng ta chưa học cách virtual machine thực sự chạy code tại runtime. Để biết điều đó, bạn sẽ phải đợi đến (hoặc nhảy ngay tới, nếu muốn) [chương tiếp theo](a-virtual-machine.html).

</aside>

Để xử lý các trường hợp như vậy, bytecode của chúng ta — giống như hầu hết các bytecode khác — cho phép instruction có <span name="operand">**operand**</span>. Chúng được lưu dưới dạng dữ liệu nhị phân ngay sau opcode trong luồng instruction và cho phép chúng ta truyền tham số để điều khiển hành vi của instruction.

<img src="image/chunks-of-bytecode/format.png" alt="OP_CONSTANT là một byte cho opcode theo sau bởi một byte cho chỉ số constant." />

Mỗi opcode sẽ quyết định nó có bao nhiêu byte operand và ý nghĩa của chúng. Ví dụ, một thao tác đơn giản như “return” có thể không có operand nào, trong khi một instruction “load local variable” cần một operand để xác định biến nào sẽ được load. Mỗi khi thêm một opcode mới vào clox, chúng ta sẽ chỉ định operand của nó trông như thế nào — tức **instruction format** của nó.

<aside name="operand">

Operand của bytecode instruction *không* giống với operand của toán tử số học. Bạn sẽ thấy khi chúng ta đến phần biểu thức rằng các giá trị operand của phép toán số học được theo dõi riêng. Operand của instruction là một khái niệm cấp thấp hơn, dùng để điều chỉnh cách mà chính instruction bytecode đó hoạt động.

</aside>

Trong trường hợp này, `OP_CONSTANT` nhận một operand dài một byte, chỉ ra constant nào sẽ được load từ mảng constant của chunk. Vì chúng ta chưa có compiler, nên ta sẽ “tự biên dịch bằng tay” một instruction trong chunk thử nghiệm.

^code main-constant (1 before, 1 after)

Chúng ta thêm chính giá trị constant đó vào constant pool của chunk. Lệnh này trả về chỉ số của constant trong mảng. Sau đó, ta ghi instruction constant, bắt đầu với opcode của nó. Tiếp theo, ta ghi operand là chỉ số constant dài một byte. Lưu ý rằng `writeChunk()` có thể ghi cả opcode lẫn operand — với hàm này, tất cả đều chỉ là các byte thô.

Nếu chạy thử ngay bây giờ, disassembler sẽ “la” lên vì nó chưa biết cách giải mã instruction mới này. Hãy sửa điều đó.

^code disassemble-constant (1 before, 1 after)

Instruction này có định dạng khác, nên ta viết một hàm trợ giúp mới để giải mã nó.

^code constant-instruction

Có nhiều việc hơn một chút ở đây. Giống như với `OP_RETURN`, ta in ra tên opcode. Sau đó, ta lấy chỉ số constant từ byte tiếp theo trong chunk. Ta in ra chỉ số đó, nhưng điều này không mấy hữu ích cho người đọc. Vì vậy, ta cũng tra luôn giá trị constant thực tế — vì constants *được* biết tại thời điểm compile — và hiển thị cả giá trị đó.

Điều này đòi hỏi một cách để in ra một clox Value. Hàm này sẽ nằm trong module "value", nên ta include nó.

^code debug-include-value (1 before, 2 after)

Trong header đó, ta khai báo:

^code print-value-h (1 before, 2 after)

Và đây là phần cài đặt:

^code print-value

Tuyệt vời, đúng không? Bạn có thể tưởng tượng rằng phần này sẽ phức tạp hơn nhiều khi chúng ta thêm dynamic typing vào Lox và có các giá trị thuộc nhiều kiểu khác nhau.

Quay lại `constantInstruction()`, phần còn lại duy nhất là giá trị trả về.

^code return-after-operand (1 before, 1 after)

Hãy nhớ rằng `disassembleInstruction()` cũng trả về một số để báo cho hàm gọi biết offset của instruction *tiếp theo*. Nếu `OP_RETURN` chỉ dài một byte, thì `OP_CONSTANT` dài hai byte — một cho opcode và một cho operand.

## Thông tin dòng (Line Information)

Chunk chứa gần như toàn bộ thông tin mà runtime cần từ mã nguồn của người dùng. Thật điên rồ khi nghĩ rằng chúng ta có thể rút gọn tất cả các lớp AST đã tạo trong jlox xuống chỉ còn một mảng byte và một mảng constants. Chỉ còn một mẩu dữ liệu chúng ta chưa có. Chúng ta cần nó, dù người dùng hy vọng sẽ không bao giờ thấy nó.

Khi xảy ra runtime error, chúng ta hiển thị cho người dùng số dòng của đoạn mã nguồn gây lỗi. Trong jlox, các số dòng này nằm trong token, và token lại được lưu trong các node AST. Giờ đây, khi đã bỏ syntax tree để dùng bytecode, chúng ta cần một giải pháp khác cho clox. Với bất kỳ instruction bytecode nào, ta cần xác định được nó được compile từ dòng nào trong mã nguồn của người dùng.

Có nhiều cách thông minh để mã hóa thông tin này. Tôi đã chọn cách <span name="side">đơn giản</span> nhất mà tôi nghĩ ra được, dù nó cực kỳ kém hiệu quả về bộ nhớ. Trong chunk, ta lưu một mảng số nguyên riêng biệt song song với bytecode. Mỗi số trong mảng là số dòng tương ứng với byte ở cùng vị trí trong bytecode. Khi xảy ra runtime error, ta tra số dòng ở cùng chỉ số với offset của instruction hiện tại trong mảng code.

<aside name="side">

Cách mã hóa “ngớ ngẩn” này lại có một điểm đúng: nó giữ thông tin dòng trong một mảng *riêng biệt* thay vì trộn vào bytecode. Vì thông tin dòng chỉ được dùng khi xảy ra runtime error, ta không muốn nó chen giữa các instruction, chiếm chỗ quý giá trong CPU cache và gây thêm cache miss khi interpreter phải bỏ qua chúng để đến opcode và operand cần thiết.

</aside>

Để cài đặt, ta thêm một mảng nữa vào Chunk.

^code chunk-lines (1 before, 1 after)

Vì nó song song hoàn toàn với mảng bytecode, ta không cần count hay capacity riêng. Mỗi khi thay đổi mảng code, ta cũng thay đổi tương ứng mảng số dòng, bắt đầu từ lúc khởi tạo.

^code chunk-null-lines (1 before, 1 after)

Và tương tự khi giải phóng:

^code chunk-free-lines (1 before, 1 after)

Khi ghi một byte code vào chunk, ta cần biết nó đến từ dòng nào trong mã nguồn, nên ta thêm một tham số nữa vào khai báo `writeChunk()`.

^code write-chunk-with-line-h (1 before, 1 after)

Và trong phần cài đặt:

^code write-chunk-with-line (1 after)

Khi cấp phát hoặc mở rộng mảng code, ta cũng làm tương tự với mảng thông tin dòng.

^code write-chunk-line (2 before, 1 after)

Cuối cùng, ta lưu số dòng vào mảng.

^code chunk-write-line (1 before, 1 after)

### Giải mã thông tin dòng (Disassembling line information)

Rồi, hãy thử áp dụng với cái chunk… thủ công của chúng ta. Trước hết, vì ta đã thêm một tham số mới vào `writeChunk()`, nên cần sửa các lời gọi hàm đó để truyền vào một số dòng — tạm thời thì cứ chọn ngẫu nhiên.

^code main-chunk-line (1 before, 2 after)

Khi có một front end thực sự, tất nhiên compiler sẽ theo dõi dòng hiện tại khi parse và truyền giá trị đó vào.

Giờ khi đã có thông tin dòng cho từng instruction, hãy tận dụng nó. Trong disassembler, việc hiển thị dòng mã nguồn mà mỗi instruction được compile từ đó sẽ rất hữu ích. Điều này giúp ta truy ngược lại code gốc khi cần tìm hiểu một đoạn bytecode nào đó đang làm gì. Sau khi in ra offset của instruction — tức số byte tính từ đầu chunk — ta sẽ hiển thị số dòng nguồn.

^code show-location (2 before, 2 after)

Các instruction bytecode thường khá chi tiết. Một dòng mã nguồn có thể compile thành cả một chuỗi instruction. Để làm rõ điều này về mặt trực quan, ta hiển thị ký tự `|` cho bất kỳ instruction nào đến từ cùng một dòng nguồn với instruction trước đó. Kết quả cho chunk viết tay của chúng ta sẽ như sau:

```text
== test chunk ==
0000  123 OP_CONSTANT         0 '1.2'
0002    | OP_RETURN
```

Chúng ta có một chunk dài ba byte. Hai byte đầu là một constant instruction load giá trị 1.2 từ constant pool của chunk. Byte đầu tiên là opcode `OP_CONSTANT` và byte thứ hai là chỉ số trong constant pool. Byte thứ ba (ở offset 2) là một return instruction dài một byte.

Trong các chương tiếp theo, chúng ta sẽ bổ sung thêm nhiều loại instruction khác. Nhưng cấu trúc cơ bản đã có, và giờ ta đã có đủ mọi thứ để biểu diễn hoàn chỉnh một đoạn code có thể execute tại runtime trong virtual machine. Nhớ cả “gia đình” các lớp AST mà ta định nghĩa trong jlox chứ? Trong clox, ta đã rút gọn xuống chỉ còn ba mảng: mảng byte code, mảng constant values, và mảng thông tin dòng để debug.

Sự rút gọn này là một lý do quan trọng khiến interpreter mới của chúng ta sẽ nhanh hơn jlox. Bạn có thể coi bytecode như một dạng “nén” gọn gàng của AST, được tối ưu mạnh mẽ cho cách interpreter sẽ giải nén và execute theo đúng thứ tự cần thiết. Trong [chương tiếp theo](a-virtual-machine.html), chúng ta sẽ thấy virtual machine thực hiện điều đó như thế nào.

<div class="challenges">

## Thử thách

1.  Cách mã hóa thông tin dòng hiện tại cực kỳ lãng phí bộ nhớ. Vì một loạt instruction thường tương ứng với cùng một dòng mã nguồn, giải pháp tự nhiên là dùng một kỹ thuật tương tự [run-length encoding](https://en.wikipedia.org/wiki/Run-length_encoding) cho số dòng.

    Hãy nghĩ ra một cách mã hóa để nén thông tin dòng cho một loạt instruction trên cùng một dòng. Thay đổi `writeChunk()` để ghi dạng nén này, và cài đặt hàm `getLine()` sao cho, khi nhận vào chỉ số của một instruction, nó trả về số dòng chứa instruction đó.

    *Gợi ý: Không cần thiết để `getLine()` quá tối ưu. Vì nó chỉ được gọi khi xảy ra runtime error, nên nó nằm ngoài đường chạy quan trọng về hiệu năng.*

2.  Vì `OP_CONSTANT` chỉ dùng một byte cho operand, một chunk chỉ có thể chứa tối đa 256 constant khác nhau. Giới hạn này đủ nhỏ để code thực tế có thể chạm tới. Ta có thể dùng hai byte hoặc nhiều hơn để lưu operand, nhưng như vậy *mọi* constant instruction sẽ tốn thêm dung lượng. Hầu hết các chunk sẽ không cần nhiều constant như vậy, nên điều đó sẽ lãng phí bộ nhớ và giảm locality trong trường hợp phổ biến chỉ để hỗ trợ trường hợp hiếm.

    Để cân bằng hai mục tiêu này, nhiều tập lệnh có nhiều instruction thực hiện cùng một thao tác nhưng với operand có kích thước khác nhau. Giữ nguyên instruction `OP_CONSTANT` một byte operand, và định nghĩa thêm một instruction `OP_CONSTANT_LONG`. Nó lưu operand dưới dạng số 24-bit, đủ thoải mái.

    Cài đặt hàm sau:

    ```c
    void writeConstant(Chunk* chunk, Value value, int line) {
      // Implement me...
    }
    ```

    Hàm này thêm `value` vào mảng constant của `chunk` rồi ghi một instruction phù hợp để load constant đó. Đồng thời thêm hỗ trợ cho disassembler để xử lý instruction `OP_CONSTANT_LONG`.

    Việc định nghĩa hai instruction dường như là “tốt cả đôi đường”. Nhưng nó có buộc ta phải hy sinh điều gì không?

3.  Hàm `reallocate()` của chúng ta dựa vào thư viện chuẩn C để cấp phát và giải phóng bộ nhớ động. `malloc()` và `free()` không phải là phép màu. Hãy tìm một vài cài đặt mã nguồn mở của chúng và giải thích cách chúng hoạt động. Chúng theo dõi byte nào đã được cấp phát và byte nào còn trống như thế nào? Cần gì để cấp phát một block bộ nhớ? Giải phóng nó? Làm sao để tối ưu hiệu năng? Chúng xử lý phân mảnh ra sao?

    *Chế độ hardcore:* Cài đặt `reallocate()` mà không gọi `realloc()`, `malloc()`, hoặc `free()`. Bạn được phép gọi `malloc()` *một lần* khi interpreter khởi động để cấp phát một block bộ nhớ lớn duy nhất, và hàm `reallocate()` của bạn sẽ quản lý vùng nhớ này. Nó sẽ chia nhỏ vùng nhớ đó thành các khối, chính là heap riêng của bạn. Nhiệm vụ của bạn là định nghĩa cách nó làm điều đó.

</div>


## Ghi chú thiết kế: Hãy kiểm thử ngôn ngữ của bạn

Chúng ta đã đi được gần nửa cuốn sách, và có một điều mà tôi vẫn chưa đề cập đến: *kiểm thử* (testing) bản cài đặt ngôn ngữ của bạn. Không phải vì kiểm thử không quan trọng. Tôi không thể nhấn mạnh đủ rằng việc có một bộ kiểm thử (test suite) tốt và toàn diện cho ngôn ngữ của bạn là cực kỳ quan trọng.

Tôi đã viết [một bộ test cho Lox](https://github.com/munificent/craftinginterpreters/tree/master/test) (bạn hoàn toàn có thể dùng cho bản cài đặt Lox của riêng mình) trước cả khi viết một chữ nào cho cuốn sách này. Những bài test đó đã tìm ra vô số lỗi trong các bản cài đặt của tôi.

Kiểm thử quan trọng với mọi phần mềm, nhưng với một ngôn ngữ lập trình thì nó còn quan trọng hơn, ít nhất là vì vài lý do sau:

*   **Người dùng kỳ vọng ngôn ngữ lập trình của họ phải cực kỳ ổn định.** Chúng ta đã quá quen với các compiler và interpreter trưởng thành, ổn định, đến mức câu “Lỗi là do code của bạn, không phải do compiler” đã trở thành [một phần ăn sâu trong văn hóa lập trình](https://blog.codinghorror.com/the-first-rule-of-programming-its-always-your-fault/). Nếu bản cài đặt ngôn ngữ của bạn có bug, người dùng sẽ phải trải qua đủ cả năm giai đoạn của sự “đau buồn” trước khi hiểu chuyện gì đang xảy ra — và bạn chắc chắn không muốn họ phải chịu đựng điều đó.

*   **Bản cài đặt ngôn ngữ là một phần mềm có mức độ liên kết nội bộ rất cao.** Một số codebase thì rộng nhưng nông. Nếu code tải file trong trình soạn thảo văn bản của bạn bị hỏng, thì — hy vọng là vậy! — nó sẽ không gây lỗi trong phần hiển thị văn bản trên màn hình. Nhưng bản cài đặt ngôn ngữ thì hẹp và sâu hơn nhiều, đặc biệt là phần lõi của interpreter xử lý ngữ nghĩa thực sự của ngôn ngữ. Điều này khiến các bug tinh vi dễ len lỏi vào do những tương tác kỳ quặc giữa các phần khác nhau của hệ thống. Cần có những bài test tốt mới “lôi” được chúng ra.

*   **Đầu vào của một bản cài đặt ngôn ngữ, theo thiết kế, là mang tính tổ hợp.** Có vô số chương trình mà người dùng có thể viết, và bản cài đặt của bạn cần chạy đúng tất cả chúng. Bạn rõ ràng không thể kiểm thử hết, nhưng bạn cần nỗ lực bao phủ càng nhiều không gian đầu vào càng tốt.

*   **Bản cài đặt ngôn ngữ thường phức tạp, thay đổi liên tục và chứa đầy tối ưu hóa.** Điều này dẫn đến code rối rắm với nhiều “góc tối” nơi bug có thể ẩn náu.

Tất cả những điều đó có nghĩa là bạn sẽ cần rất nhiều bài test. Nhưng *test gì*? Những dự án tôi từng thấy thường tập trung vào các “bài test ngôn ngữ” end-to-end. Mỗi bài test là một chương trình viết bằng chính ngôn ngữ đó, kèm theo output hoặc lỗi mong đợi. Sau đó, bạn có một test runner để đưa chương trình test qua bản cài đặt ngôn ngữ của mình và kiểm tra xem nó có hoạt động đúng như mong đợi không. Viết test bằng chính ngôn ngữ đó có vài lợi thế:

*   Các bài test không phụ thuộc vào bất kỳ API hay quyết định kiến trúc nội bộ nào của bản cài đặt. Điều này cho phép bạn tái cấu trúc hoặc viết lại các phần của interpreter hoặc compiler mà không cần sửa hàng loạt bài test.

*   Bạn có thể dùng cùng một bộ test cho nhiều bản cài đặt khác nhau của ngôn ngữ.

*   Các bài test thường ngắn gọn, dễ đọc và dễ bảo trì vì chúng chỉ là các script trong ngôn ngữ của bạn.

Tất nhiên, không phải mọi thứ đều “màu hồng”:

*   Test end-to-end giúp bạn xác định *có* bug hay không, nhưng không cho biết *bug ở đâu*. Sẽ khó hơn để tìm ra đoạn code gây lỗi trong bản cài đặt, vì tất cả những gì test nói với bạn là output không đúng.

*   Việc tạo ra một chương trình hợp lệ để “chạm” tới một góc khuất nào đó của bản cài đặt có thể rất mất công. Điều này đặc biệt đúng với các compiler tối ưu hóa cao, nơi bạn có thể phải viết code vòng vèo để đảm bảo rơi đúng vào nhánh tối ưu hóa mà bug đang ẩn.

*   Chi phí khởi động interpreter, parse, compile và chạy mỗi script test có thể cao. Với một bộ test lớn — mà bạn *nên* có, nhớ nhé — điều đó có thể đồng nghĩa với việc phải chờ rất lâu để chạy xong toàn bộ.

Tôi có thể nói tiếp nữa, nhưng không muốn biến phần này thành một bài thuyết giảng. Tôi cũng không giả vờ là chuyên gia về *cách* kiểm thử ngôn ngữ. Tôi chỉ muốn bạn thật sự ghi nhớ tầm quan trọng của việc *phải* kiểm thử ngôn ngữ của mình. Nghiêm túc đấy. Hãy kiểm thử ngôn ngữ của bạn. Sau này bạn sẽ cảm ơn tôi.