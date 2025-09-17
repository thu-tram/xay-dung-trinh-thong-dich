
> I wanna, I wanna,<br />
> I wanna, I wanna,<br />
> I wanna be trash.<br />
>
> <cite>The Whip, &ldquo;Trash&rdquo;</cite>

Chúng ta gọi Lox là một ngôn ngữ “bậc cao” vì nó giúp lập trình viên không phải bận tâm đến những chi tiết không liên quan trực tiếp đến vấn đề họ đang giải quyết. Người dùng trở thành “nhà điều hành”, đưa ra các mục tiêu trừu tượng và để chiếc máy tính “thấp hèn” tự tìm cách thực hiện.

Cấp phát bộ nhớ động là một ứng viên hoàn hảo để tự động hóa. Nó cần thiết cho một chương trình hoạt động, tẻ nhạt nếu làm thủ công, và vẫn dễ mắc lỗi. Những sai sót không tránh khỏi có thể gây hậu quả nghiêm trọng, dẫn đến crash, hỏng dữ liệu trong bộ nhớ, hoặc vi phạm bảo mật. Đây là loại công việc vừa rủi ro vừa nhàm chán mà máy móc làm tốt hơn con người.

Đó là lý do Lox là một **managed language**, nghĩa là implementation của ngôn ngữ sẽ quản lý việc cấp phát và giải phóng bộ nhớ thay cho người dùng. Khi người dùng thực hiện một thao tác cần bộ nhớ động, VM sẽ tự động cấp phát. Lập trình viên không bao giờ phải lo việc giải phóng. Máy sẽ đảm bảo bất kỳ vùng nhớ nào chương trình đang dùng sẽ tồn tại chừng nào còn cần thiết.

Lox tạo ra ảo giác rằng máy tính có bộ nhớ vô hạn. Người dùng có thể cấp phát, cấp phát, và cấp phát mà không bao giờ phải nghĩ xem tất cả những byte đó đến từ đâu. Tất nhiên, máy tính hiện tại *không* có bộ nhớ vô hạn. Vì vậy, cách mà các ngôn ngữ managed duy trì ảo giác này là âm thầm thu hồi lại những vùng nhớ mà chương trình không còn cần nữa. Thành phần làm việc này được gọi là **garbage <span name="recycle">collector</span>**.

<aside name="recycle">

Thực ra, “tái chế” sẽ là một phép ẩn dụ hay hơn. GC không *vứt bỏ* bộ nhớ, mà thu hồi nó để tái sử dụng cho dữ liệu mới. Nhưng các ngôn ngữ managed ra đời trước cả Ngày Trái Đất, nên những người phát minh đã dùng phép so sánh mà họ biết.

<img src="image/garbage-collection/recycle.png" class="above" alt="Một thùng tái chế đầy các bit dữ liệu." />

</aside>

## Reachability

Điều này dẫn đến một câu hỏi khó bất ngờ: làm sao VM biết được vùng nhớ nào *không* cần nữa? Bộ nhớ chỉ cần thiết nếu nó sẽ được đọc trong tương lai, nhưng trừ khi có cỗ máy thời gian, làm sao implementation biết được chương trình *sẽ* chạy đoạn code nào và dùng dữ liệu nào? Tiết lộ trước: VM không thể du hành thời gian. Thay vào đó, ngôn ngữ đưa ra một giả định <span name="conservative">bảo thủ</span>: nó coi một vùng nhớ vẫn còn được sử dụng nếu *có thể* sẽ được đọc trong tương lai.

<aside name="conservative">

Tôi dùng “bảo thủ” ở nghĩa chung. Có một khái niệm “conservative garbage collector” mang nghĩa cụ thể hơn. Tất cả GC đều “bảo thủ” ở chỗ chúng giữ vùng nhớ sống nếu nó *có thể* được truy cập, thay vì có một quả cầu tiên tri Magic 8-Ball để biết chính xác dữ liệu nào *sẽ* được truy cập.

**Conservative GC** là một loại collector đặc biệt, coi bất kỳ vùng nhớ nào là một con trỏ nếu giá trị trong đó trông giống một địa chỉ. Ngược lại, **precise GC** — loại mà chúng ta sẽ implement — biết chính xác từ nào trong bộ nhớ là con trỏ và từ nào lưu các loại giá trị khác như số hoặc chuỗi.

</aside>

Nghe có vẻ *quá* bảo thủ. Chẳng phải *bất kỳ* bit bộ nhớ nào cũng có thể được đọc sao? Thực ra là không, ít nhất là trong một ngôn ngữ an toàn bộ nhớ như Lox. Ví dụ:

```lox
var a = "first value";
a = "updated";
// GC here.
print a;
```

Giả sử ta chạy GC sau khi lệnh gán ở dòng thứ hai hoàn tất. Chuỗi `"first value"` vẫn nằm trong bộ nhớ, nhưng chương trình của người dùng không còn cách nào để truy cập nó. Khi `a` được gán lại, chương trình mất mọi tham chiếu tới chuỗi đó. Ta có thể giải phóng nó an toàn. Một giá trị được gọi là **reachable** nếu có cách nào đó để chương trình người dùng tham chiếu tới nó. Ngược lại, như chuỗi `"first value"` ở đây, nó là **unreachable**.

Nhiều giá trị có thể được VM truy cập trực tiếp. Xem ví dụ:

```lox
var global = "string";
{
  var local = "another";
  print global + local;
}
```

Tạm dừng chương trình ngay sau khi hai chuỗi được nối nhưng trước khi câu lệnh `print` chạy. VM có thể tìm `"string"` bằng cách duyệt bảng biến global và tìm mục cho `global`. Nó có thể tìm `"another"` bằng cách duyệt value stack và lấy slot của biến local `local`. Nó thậm chí có thể tìm chuỗi tạm `"stringanother"` vì giá trị tạm này cũng đang nằm trên stack của VM tại thời điểm ta tạm dừng.

Tất cả những giá trị này được gọi là **root**. Root là bất kỳ object nào mà VM có thể truy cập trực tiếp mà không cần thông qua tham chiếu trong object khác. Hầu hết root là biến global hoặc trên stack, nhưng như ta sẽ thấy, còn có vài nơi khác VM lưu tham chiếu tới object mà nó có thể tìm được.

Các giá trị khác có thể được tìm thấy bằng cách đi qua một tham chiếu bên trong giá trị khác. <span name="class">Field</span> trên instance của class là trường hợp rõ ràng nhất, nhưng ta chưa có chúng. Ngay cả khi chưa có, VM của ta vẫn có các tham chiếu gián tiếp. Xem ví dụ:

<aside name="class">

Chúng ta sẽ sớm gặp lại [trong phần này](classes-and-instances.html)!

</aside>

```lox
fun makeClosure() {
  var a = "data";

  fun f() { print a; }
  return f;
}

{
  var closure = makeClosure();
  // GC here.
  closure();
}
```

Giả sử ta tạm dừng chương trình ở dòng được đánh dấu và chạy garbage collector. Khi collector xong và chương trình tiếp tục, nó sẽ gọi closure, và closure sẽ in ra `"data"`. Vậy collector cần *không* được giải phóng chuỗi đó. Nhưng đây là stack khi ta tạm dừng:

<img src="image/garbage-collection/stack.png" alt="Stack, chỉ chứa script và closure." />

Chuỗi `"data"` không hề có trên đó. Nó đã được đưa ra khỏi stack và lưu trong closed upvalue mà closure sử dụng. Bản thân closure thì nằm trên stack. Nhưng để tới được chuỗi, ta cần lần theo closure và mảng upvalue của nó. Vì chương trình người dùng *có thể* làm vậy, tất cả các object có thể truy cập gián tiếp này cũng được coi là reachable.

<img src="image/garbage-collection/reachable.png" class="wide" alt="Tất cả object được tham chiếu từ closure, và đường dẫn tới chuỗi 'data' từ stack." />

Điều này cho ta một định nghĩa quy nạp về reachability:

*   Tất cả root đều reachable.

*   Bất kỳ object nào được tham chiếu từ một object reachable cũng là reachable.

Đây là những giá trị vẫn “sống” và cần được giữ lại trong bộ nhớ. Bất kỳ giá trị nào *không* thỏa định nghĩa này đều có thể bị collector thu hồi. Cặp quy tắc đệ quy này gợi ý một thuật toán đệ quy mà ta có thể dùng để giải phóng bộ nhớ không cần thiết:

1.  Bắt đầu từ các root, duyệt qua các tham chiếu object để tìm toàn bộ tập hợp object reachable.

2.  Giải phóng tất cả object *không* nằm trong tập hợp đó.

Hiện nay có rất <span name="handbook">nhiều</span> thuật toán garbage collection khác nhau, nhưng tất cả đều tuân theo cấu trúc chung này. Một số có thể xen kẽ hoặc trộn các bước, nhưng hai thao tác cơ bản vẫn ở đó. Chúng khác nhau chủ yếu ở *cách* thực hiện từng bước.

<aside name="handbook">

Nếu bạn muốn tìm hiểu thêm về các thuật toán GC khác,  
[*The Garbage Collection Handbook*](http://gchandbook.org/) (Jones và cộng sự) là tài liệu tham khảo kinh điển. Với một cuốn sách dày về một chủ đề vừa sâu vừa hẹp như vậy, nó lại khá thú vị để đọc. Hoặc có thể tôi có một khái niệm hơi khác về “thú vị”.

</aside>


## Mark-Sweep Garbage Collection

Ngôn ngữ managed đầu tiên là Lisp, ngôn ngữ “bậc cao” thứ hai được phát minh, ngay sau Fortran. John McCarthy từng cân nhắc việc dùng quản lý bộ nhớ thủ công hoặc reference counting, nhưng <span name="procrastination">cuối cùng</span> đã chọn (và đặt tên) cho garbage collection — khi chương trình hết bộ nhớ, nó sẽ quay lại tìm những vùng nhớ không dùng nữa để thu hồi.

<aside name="procrastination">

Trong “History of Lisp” của John McCarthy, ông ghi chú: “Khi chúng tôi quyết định dùng garbage collection, việc implement thực tế có thể trì hoãn, vì lúc đó chỉ mới làm các ví dụ nhỏ.” Việc chúng ta trì hoãn thêm GC vào clox cũng là đi theo bước chân của những bậc tiền bối.

</aside>

Ông đã thiết kế thuật toán garbage collection đầu tiên và đơn giản nhất, gọi là **mark-and-sweep** hoặc **mark-sweep**. Mô tả của nó chỉ gói gọn trong ba đoạn ngắn trong bài báo đầu tiên về Lisp. Dù đã cũ và đơn giản, cùng một thuật toán nền tảng này vẫn là cơ sở cho nhiều bộ quản lý bộ nhớ hiện đại. Một số lĩnh vực của khoa học máy tính dường như là bất biến theo thời gian.

Đúng như tên gọi, mark-sweep hoạt động qua hai giai đoạn:

*   **Marking:** Bắt đầu từ các root và duyệt hoặc <span name="trace">*trace*</span> qua tất cả object mà các root đó tham chiếu tới. Đây là một dạng duyệt đồ thị kinh điển qua tất cả object reachable. Mỗi lần thăm một object, ta sẽ *đánh dấu* nó theo một cách nào đó. (Các implementation khác nhau ở cách lưu dấu này.)

*   **Sweeping:** Khi giai đoạn mark hoàn tất, mọi object reachable trong heap đã được đánh dấu. Điều đó có nghĩa là bất kỳ object nào không được đánh dấu đều là unreachable và sẵn sàng để thu hồi. Ta sẽ duyệt qua tất cả object chưa được đánh dấu và giải phóng từng cái.

Nó trông như thế này:

<img src="image/garbage-collection/mark-sweep.png" class="wide" alt="Bắt đầu từ một đồ thị object, trước tiên đánh dấu các object reachable, quét bỏ phần còn lại, và cuối cùng chỉ còn lại các object reachable." />

<aside name="trace">

**Tracing garbage collector** là bất kỳ thuật toán nào duyệt qua đồ thị các tham chiếu object. Điều này trái ngược với reference counting, vốn dùng chiến lược khác để theo dõi object reachable.

</aside>

Đó chính là thứ chúng ta sẽ implement. Mỗi khi quyết định đã đến lúc thu hồi một số byte, ta sẽ trace mọi thứ và đánh dấu tất cả object reachable, giải phóng những gì không được đánh dấu, rồi tiếp tục chạy chương trình của người dùng.

### Collecting garbage

Cả chương này xoay quanh việc implement một <span name="one">hàm</span> này:

<aside name="one">

Tất nhiên, chúng ta sẽ thêm một loạt hàm helper nữa.

</aside>

^code collect-garbage-h (1 before, 1 after)

Chúng ta sẽ tiến dần tới bản implement đầy đủ, bắt đầu từ phần khung rỗng này:

^code collect-garbage

Câu hỏi đầu tiên bạn có thể đặt ra là: Hàm này được gọi khi nào? Hóa ra đây là một câu hỏi tinh tế mà ta sẽ dành thời gian bàn ở phần sau của chương. Còn bây giờ, ta sẽ tạm bỏ qua và nhân tiện xây một công cụ chẩn đoán hữu ích.

^code define-stress-gc (1 before, 2 after)

Ta sẽ thêm một chế độ “stress test” tùy chọn cho garbage collector. Khi flag này được bật, GC sẽ chạy thường xuyên nhất có thể. Điều này, tất nhiên, là thảm họa cho hiệu năng. Nhưng nó rất hữu ích để lộ ra các bug quản lý bộ nhớ chỉ xảy ra khi GC được kích hoạt đúng thời điểm. Nếu *mọi* thời điểm đều kích hoạt GC, bạn sẽ dễ tìm ra những bug đó.

^code call-collect (1 before, 1 after)

Mỗi khi ta gọi `reallocate()` để xin thêm bộ nhớ, ta sẽ buộc chạy một lần collect. Lệnh if ở đây là vì `reallocate()` cũng được gọi khi giải phóng hoặc thu nhỏ một allocation. Ta không muốn kích hoạt GC cho những trường hợp đó — đặc biệt vì bản thân GC cũng sẽ gọi `reallocate()` để giải phóng bộ nhớ.

Collect ngay trước khi <span name="demand">allocation</span> là cách kinh điển để tích hợp GC vào VM. Bạn vốn đã gọi vào memory manager, nên đây là chỗ dễ dàng để móc thêm code. Hơn nữa, allocation là thời điểm duy nhất bạn thực sự *cần* một ít bộ nhớ trống để tái sử dụng. Nếu bạn *không* dùng allocation để kích hoạt GC, bạn sẽ phải đảm bảo mọi nơi trong code có thể lặp và cấp phát bộ nhớ cũng có cách kích hoạt collector. Nếu không, VM có thể rơi vào trạng thái “đói” bộ nhớ — cần thêm nhưng không bao giờ collect.

<aside name="demand">

Các collector tinh vi hơn có thể chạy trên một thread riêng hoặc được xen kẽ định kỳ trong quá trình execute chương trình — thường ở ranh giới function call hoặc khi xảy ra một backward jump.

</aside>

### Debug logging

Nhân nói về chẩn đoán, hãy thêm một chút nữa. Một thách thức thực sự với garbage collector là chúng rất “mờ”. Từ trước đến giờ, ta vẫn chạy nhiều chương trình Lox ngon lành mà *không hề* có GC. Khi thêm GC vào, làm sao biết nó đang làm gì hữu ích? Liệu ta chỉ biết khi viết chương trình tiêu tốn hàng đống bộ nhớ? Làm sao debug chuyện đó?

Một cách đơn giản để soi vào bên trong GC là thêm logging.

^code define-log-gc (1 before, 2 after)

Khi bật flag này, clox sẽ in thông tin ra console mỗi khi nó làm gì đó với bộ nhớ động.

Ta cần thêm vài include.

^code debug-log-includes (1 before, 2 after)

Ta chưa có collector, nhưng có thể bắt đầu thêm logging ngay bây giờ. Ta sẽ muốn biết khi nào một lần collect bắt đầu.

^code log-before-collect (1 before, 1 after)

Cuối cùng, ta sẽ log thêm vài thao tác khác trong quá trình collect, nên cũng cần biết khi nào “buổi diễn” kết thúc.

^code log-after-collect (2 before, 1 after)

Ta chưa có code cho collector, nhưng đã có các hàm allocate và free, nên có thể gắn logging vào đó ngay.

^code debug-log-allocate (1 before, 1 after)

Và khi vòng đời của một object kết thúc:

^code log-free-object (1 before, 1 after)

Với hai flag này, ta sẽ thấy được tiến trình khi làm nốt phần còn lại của chương.

## Đánh dấu các Root

Các object nằm rải rác khắp heap như những vì sao trên bầu trời đêm thăm thẳm.  
Một reference từ object này sang object khác tạo thành một kết nối, và những “chòm sao” này chính là đồ thị mà giai đoạn **mark** sẽ duyệt qua. Việc đánh dấu bắt đầu từ các **root**.

^code call-mark-roots (3 before, 2 after)

Hầu hết các root là biến cục bộ hoặc biến tạm thời nằm ngay trong stack của VM,  
nên chúng ta bắt đầu bằng cách duyệt qua stack đó.

^code mark-roots

Để đánh dấu một giá trị Lox, ta dùng hàm mới này:

^code mark-value-h (1 before, 1 after)

Phần hiện thực của nó nằm ở đây:

^code mark-value

Một số giá trị Lox — số, Boolean và `nil` — được lưu trực tiếp bên trong `Value` và không cần cấp phát trên heap.  
Bộ **garbage collector** hoàn toàn không cần quan tâm đến chúng, nên việc đầu tiên ta làm là đảm bảo giá trị đó thực sự là một object trên heap.  
Nếu đúng vậy, công việc chính sẽ diễn ra trong hàm này:

^code mark-object-h (1 before, 1 after)

Hàm này được định nghĩa ở đây:

^code mark-object

Việc kiểm tra `NULL` là không cần thiết khi gọi từ `markValue()`.  
Một giá trị Lox thuộc loại Obj sẽ luôn có một con trỏ hợp lệ.  
Nhưng sau này, chúng ta sẽ gọi hàm này trực tiếp từ những đoạn code khác, và ở một số nơi, object được trỏ tới có thể là tùy chọn (optional).

Giả sử ta có một object hợp lệ, ta sẽ đánh dấu nó bằng cách bật một cờ (flag).  
Trường dữ liệu mới này nằm trong struct header `Obj` mà tất cả object đều chia sẻ.

^code is-marked-field (1 before, 1 after)

Mỗi object mới khi được tạo ra đều bắt đầu ở trạng thái **chưa được đánh dấu**,  
vì ta chưa xác định được nó có thể truy cập được (reachable) hay không.

^code init-is-marked (1 before, 2 after)

Trước khi đi xa hơn, hãy thêm một chút logging vào `markObject()`.

^code log-mark-object (2 before, 1 after)

Nhờ vậy, ta có thể thấy giai đoạn **mark** đang làm gì.  
Việc đánh dấu stack sẽ xử lý các biến cục bộ và biến tạm thời.  
Nguồn root chính còn lại là các biến toàn cục.

^code mark-globals (2 before, 1 after)

Chúng được lưu trong một hash table thuộc sở hữu của VM,  
nên ta sẽ khai báo thêm một hàm helper để đánh dấu tất cả object trong một bảng.

^code mark-table-h (2 before, 2 after)

Ta hiện thực hàm này trong module `"table"` tại đây:

^code mark-table

Rất đơn giản: ta duyệt qua mảng entry, với mỗi entry ta đánh dấu giá trị của nó.  
Ta cũng đánh dấu cả key string của mỗi entry vì GC cũng quản lý các string này.

### Những root ít hiển nhiên hơn

Những phần trên bao quát các root mà ta thường nghĩ tới — các giá trị rõ ràng là có thể truy cập được vì chúng được lưu trong các biến mà chương trình của người dùng có thể thấy.  
Nhưng VM cũng có một vài “ngóc ngách” riêng, nơi nó cất giữ các reference tới những giá trị mà nó truy cập trực tiếp.

Hầu hết trạng thái của lời gọi hàm nằm trong value stack,  
nhưng VM còn duy trì một stack riêng của các `CallFrame`.  
Mỗi `CallFrame` chứa một con trỏ tới closure đang được gọi.  
VM dùng các con trỏ này để truy cập hằng số và upvalue,  
nên những closure này cũng cần được giữ lại.

^code mark-closures (1 before, 2 after)

Nói đến upvalue, danh sách **open upvalue** cũng là một tập hợp giá trị khác mà VM có thể truy cập trực tiếp.

^code mark-open-upvalues (3 before, 2 after)

Hãy nhớ rằng quá trình collect có thể bắt đầu trong *bất kỳ* lần allocation nào. Những allocation này không chỉ xảy ra khi chương trình của người dùng đang chạy. Bản thân compiler cũng định kỳ lấy bộ nhớ từ heap cho literal và constant table. Nếu GC chạy khi ta đang ở giữa quá trình compile, thì bất kỳ giá trị nào mà compiler truy cập trực tiếp cũng cần được coi là root.

Để giữ cho module compiler tách biệt rõ ràng với phần còn lại của VM, ta sẽ làm điều đó trong một hàm riêng.

^code call-mark-compiler-roots (1 before, 1 after)

Hàm này được khai báo ở đây:

^code mark-compiler-roots-h (1 before, 2 after)

Điều này có nghĩa là module "memory" cần include thêm.

^code memory-include-compiler (2 before, 1 after)

Và phần định nghĩa nằm trong module "compiler".

^code mark-compiler-roots

May mắn là compiler không giữ lại quá nhiều giá trị. Object duy nhất nó dùng là ObjFunction mà nó đang compile. Vì function declaration có thể lồng nhau, compiler có một linked list các ObjFunction đó và ta sẽ duyệt toàn bộ danh sách.

Vì module "compiler" gọi `markObject()`, nó cũng cần include thêm.

^code compiler-include-memory (1 before, 1 after)

Đó là tất cả các root. Sau khi chạy bước này, mọi object mà VM — cả runtime và compiler — có thể truy cập *mà không* cần đi qua object khác đều đã được set mark bit.

## Tracing Object References

Bước tiếp theo trong quá trình marking là lần theo đồ thị các tham chiếu giữa các object để tìm các giá trị reachable gián tiếp. Chúng ta chưa có instance với field, nên chưa có nhiều object chứa tham chiếu, nhưng vẫn có <span name="some">một vài</span>. Cụ thể, ObjClosure có danh sách ObjUpvalue mà nó đóng gói, cũng như tham chiếu tới ObjFunction gốc mà nó bao bọc. ObjFunction, đến lượt nó, có constant table chứa tham chiếu tới tất cả literal được tạo trong phần thân function. Chừng đó là đủ để tạo ra một mạng lưới object khá phức tạp cho collector duyệt qua.

<aside name="some">

Tôi đặt chương này vào đúng chỗ này trong sách *chính vì* giờ chúng ta đã có closure, thứ mang lại cho garbage collector những object thú vị để xử lý.

</aside>

Giờ là lúc implement việc duyệt này. Ta có thể duyệt breadth-first, depth-first, hoặc theo thứ tự khác. Vì ta chỉ cần tìm *tập hợp* tất cả object reachable, thứ tự duyệt <span name="dfs">hầu như</span> không quan trọng.

<aside name="dfs">

Tôi nói “hầu như” vì một số garbage collector sẽ di chuyển object theo thứ tự chúng được duyệt, nên thứ tự duyệt quyết định object nào nằm cạnh nhau trong bộ nhớ. Điều này ảnh hưởng đến hiệu năng vì CPU tận dụng tính cục bộ để quyết định vùng nhớ nào sẽ được nạp trước vào cache.

Ngay cả khi thứ tự duyệt có quan trọng, cũng không rõ thứ tự nào là *tốt nhất*. Rất khó để xác định object nào sẽ được dùng trong tương lai, nên GC khó biết thứ tự nào sẽ giúp hiệu năng tốt hơn.

</aside>

### The tricolor abstraction

Khi collector di chuyển qua đồ thị object, ta cần đảm bảo nó không bị mất dấu vị trí hoặc bị kẹt trong vòng lặp. Điều này đặc biệt quan trọng với các implementation nâng cao như incremental GC, vốn xen kẽ việc marking với việc chạy các phần của chương trình người dùng. Collector cần có khả năng tạm dừng và tiếp tục từ đúng chỗ nó dừng lại.

Để giúp chúng ta — những con người “não mềm” — dễ hình dung quá trình phức tạp này, các hacker VM đã nghĩ ra một phép ẩn dụ gọi là <span name="color"></span>**tricolor abstraction**. Mỗi object có một “màu” khái niệm để theo dõi trạng thái của nó và công việc còn lại cần làm.

<aside name="color">

Các thuật toán garbage collection nâng cao thường thêm các màu khác vào mô hình này. Tôi từng thấy nhiều sắc độ xám, thậm chí cả màu tím trong một số thiết kế. Bài báo về collector màu puce-chartreuse-fuchsia-malachite của tôi thì, tiếc là, không được chấp nhận đăng.

</aside>

*   **<img src="image/garbage-collection/white.png" alt="A white circle."
    class="dot" /> White:** Ở đầu quá trình garbage collection, mọi object đều màu trắng. Màu này nghĩa là ta chưa hề chạm tới hoặc xử lý object đó.

*   **<img src="image/garbage-collection/gray.png" alt="A gray circle."
    class="dot" /> Gray:** Trong quá trình marking, khi ta vừa chạm tới một object, ta đổi nó sang màu xám. Màu này nghĩa là ta biết bản thân object đó là reachable và không nên bị collect. Nhưng ta vẫn chưa lần *qua* nó để xem nó tham chiếu tới *các* object nào khác. Trong thuật toán đồ thị, đây là *worklist* — tập hợp các object ta biết nhưng chưa xử lý.

*   **<img src="image/garbage-collection/black.png" alt="A black circle."
    class="dot" /> Black:** Khi ta lấy một object màu xám và đánh dấu tất cả object mà nó tham chiếu, ta sẽ đổi object màu xám đó thành màu đen. Màu này nghĩa là giai đoạn mark đã xử lý xong object đó.

Theo mô hình này, quá trình marking sẽ như sau:

1.  Bắt đầu với tất cả object màu trắng.

2.  Tìm tất cả root và đánh dấu chúng thành màu xám.

3.  Lặp lại chừng nào vẫn còn object màu xám:

    1.  Chọn một object màu xám. Đổi bất kỳ object màu trắng nào mà nó tham chiếu thành màu xám.

    2.  Đánh dấu object màu xám ban đầu thành màu đen.

Tôi thấy việc hình dung sẽ giúp dễ hiểu hơn. Bạn có một mạng lưới object với các tham chiếu giữa chúng. Ban đầu, tất cả là những chấm trắng nhỏ. Ở bên ngoài có vài cạnh đi vào từ VM trỏ tới các root. Các root đó chuyển sang màu xám. Rồi mỗi “hàng xóm” của object màu xám lại chuyển sang màu xám, trong khi bản thân object đó chuyển sang màu đen. Hiệu ứng tổng thể là một “làn sóng” màu xám lan qua đồ thị, để lại một vùng các object màu đen reachable phía sau. Các object unreachable thì không bị làn sóng chạm tới và vẫn giữ màu trắng.

<img src="image/garbage-collection/tricolor-trace.png" class="wide" alt="A gray wavefront working through a graph of nodes." />

Ở <span name="invariant">cuối</span> quá trình, bạn sẽ còn lại một “biển” các object màu đen đã được đánh dấu reachable, xen kẽ là những “hòn đảo” object màu trắng có thể được quét và giải phóng. Khi các object unreachable đã được giải phóng, những object còn lại — tất cả đều màu đen — sẽ được reset về màu trắng cho chu kỳ garbage collection tiếp theo.

<aside name="invariant">

Lưu ý rằng ở mọi bước của quá trình này, không có node màu đen nào trỏ tới node màu trắng. Thuộc tính này được gọi là **tricolor invariant**. Quá trình duyệt giữ nguyên invariant này để đảm bảo không có object reachable nào bị collect nhầm.

</aside>

### Danh sách công việc cho các object màu xám

Trong phần implement của chúng ta, các root đã được đánh dấu. Tất cả chúng đều màu xám. Bước tiếp theo là bắt đầu chọn chúng và duyệt qua các tham chiếu của chúng. Nhưng hiện tại ta không có cách dễ dàng nào để tìm chúng. Ta đã set một field trên object, nhưng chỉ vậy thôi. Ta không muốn phải duyệt toàn bộ danh sách object để tìm những object có field đó được set.

Thay vào đó, ta sẽ tạo một worklist riêng để theo dõi tất cả object màu xám. Khi một object chuyển sang màu xám, ngoài việc set mark field, ta cũng sẽ thêm nó vào worklist.

^code add-to-gray-stack (1 before, 1 after)

Ta có thể dùng bất kỳ cấu trúc dữ liệu nào cho phép thêm và lấy phần tử dễ dàng. Tôi chọn stack vì nó đơn giản nhất để implement bằng dynamic array trong C. Nó hoạt động gần giống các dynamic array khác mà ta đã xây trong Lox, *ngoại trừ* việc nó gọi hàm `realloc()` của *hệ thống* chứ không phải wrapper `reallocate()` của chúng ta. Bộ nhớ cho gray stack *không* được quản lý bởi garbage collector. Ta không muốn việc mở rộng gray stack trong lúc GC chạy lại khiến GC đệ quy khởi động một GC mới. Điều đó có thể “xé toạc” kết cấu không-thời gian.

Ta sẽ tự quản lý bộ nhớ của nó một cách tường minh. VM sở hữu gray stack.

^code vm-gray-stack (1 before, 1 after)

Ban đầu nó rỗng.

^code init-gray-stack (1 before, 2 after)

Và ta cần giải phóng nó khi VM tắt.

^code free-gray-stack (2 before, 1 after)

<span name="robust">Chúng ta</span> chịu hoàn toàn trách nhiệm với mảng này. Điều đó bao gồm cả việc xử lý khi allocation thất bại. Nếu ta không thể tạo hoặc mở rộng gray stack, thì ta không thể hoàn tất garbage collection. Đây là tin xấu cho VM, nhưng may mắn là hiếm khi xảy ra vì gray stack thường khá nhỏ. Sẽ tốt hơn nếu làm gì đó “êm” hơn, nhưng để code trong sách đơn giản, ta chỉ việc abort.

<aside name="robust">

Để robust hơn, ta có thể cấp phát một khối bộ nhớ “dự phòng” khi khởi động VM. Nếu allocation cho gray stack thất bại, ta giải phóng khối dự phòng này và thử lại. Điều đó có thể cho ta đủ khoảng trống trên heap để tạo gray stack, hoàn tất GC, và giải phóng thêm bộ nhớ.

</aside>

^code exit-gray-stack (2 before, 1 after)

### Xử lý các object màu xám

OK, giờ khi đã đánh dấu xong các root, ta vừa set một loạt field vừa lấp đầy worklist bằng các object cần xử lý. Đến lúc chuyển sang giai đoạn tiếp theo.

^code call-trace-references (1 before, 2 after)

Đây là phần implement:

^code trace-references

Nó gần giống nhất với thuật toán dạng văn bản mà bạn có thể hình dung. Cho đến khi stack rỗng, ta liên tục lấy ra các object màu xám, duyệt qua các tham chiếu của chúng, rồi đánh dấu chúng thành màu đen. Việc duyệt tham chiếu của một object có thể phát hiện ra các object màu trắng mới, đánh dấu chúng thành màu xám và thêm vào stack. Vậy nên hàm này cứ đan xen giữa việc biến object trắng thành xám và xám thành đen, dần dần đẩy “làn sóng” tiến về phía trước.

Đây là nơi ta duyệt tham chiếu của một object đơn lẻ:

^code blacken-object

Mỗi loại object <span name="leaf">kind</span> có các field khác nhau có thể tham chiếu tới object khác, nên ta cần một đoạn code riêng cho từng loại. Ta bắt đầu với loại dễ — string và native function object không chứa tham chiếu ra ngoài nên không có gì để duyệt.

<aside name="leaf">

Một tối ưu hóa đơn giản ta có thể làm trong `markObject()` là bỏ qua việc thêm string và native function vào gray stack ngay từ đầu vì ta biết chúng không cần xử lý. Thay vào đó, chúng có thể chuyển thẳng từ trắng sang đen.

</aside>

Lưu ý rằng ta không set bất kỳ trạng thái nào trong chính object đã duyệt. Không có mã hóa trực tiếp nào cho “màu đen” trong trạng thái của object. Một object màu đen là bất kỳ object nào có field `isMarked` được <span name="field">set</span> và không còn nằm trong gray stack.

<aside name="field">

Bạn có thể tự hỏi tại sao ta lại cần field `isMarked`. Rồi sẽ rõ thôi, bạn của tôi.

</aside>

Giờ hãy bắt đầu thêm các loại object khác. Đơn giản nhất là upvalue.

^code blacken-upvalue (2 before, 1 after)

Khi một upvalue được đóng, nó chứa tham chiếu tới giá trị đã đóng. Vì giá trị này không còn trên stack, ta cần đảm bảo lần theo tham chiếu tới nó từ upvalue.

Tiếp theo là function.

^code blacken-function (1 before, 1 after)

Mỗi function có tham chiếu tới một ObjString chứa tên function. Quan trọng hơn, function có một constant table chứa đầy tham chiếu tới các object khác. Ta sẽ trace tất cả chúng bằng helper này:

^code mark-array

Loại object cuối cùng mà ta có hiện tại — sẽ thêm nữa ở các chương sau — là closure.

^code blacken-closure (1 before, 1 after)

Mỗi closure có tham chiếu tới function gốc mà nó bao bọc, cũng như một mảng con trỏ tới các upvalue mà nó capture. Ta trace tất cả chúng.

Đó là cơ chế cơ bản để xử lý một object màu xám, nhưng còn hai chi tiết cần hoàn thiện. Đầu tiên là logging.

^code log-blacken-object (1 before, 1 after)

Cách này giúp ta quan sát việc tracing lan tỏa qua đồ thị object. Nhân tiện, tôi nói là *đồ thị*. Các tham chiếu giữa object là có hướng, nhưng điều đó không có nghĩa là chúng *phi chu trình*! Hoàn toàn có thể tồn tại các vòng lặp object. Khi điều đó xảy ra, ta cần đảm bảo collector không bị kẹt trong vòng lặp vô hạn khi liên tục thêm lại cùng một chuỗi object vào gray stack.

Cách xử lý rất đơn giản.

^code check-is-marked (1 before, 1 after)

Nếu object đã được đánh dấu, ta sẽ không đánh dấu lại và do đó không thêm nó vào gray stack. Điều này đảm bảo rằng một object đã màu xám sẽ không bị thêm trùng lặp, và một object màu đen sẽ không bị chuyển ngược lại thành xám. Nói cách khác, nó giữ cho “làn sóng” chỉ tiến qua các object màu trắng.


## Quét các object không dùng

Khi vòng lặp trong `traceReferences()` kết thúc, chúng ta đã xử lý hết mọi object có thể “chạm tới”. Gray stack trống rỗng, và mọi object trong heap hoặc là đen hoặc là trắng. Các object đen là reachable, và ta muốn giữ chúng lại. Bất cứ thứ gì còn trắng đều chưa hề bị trace chạm tới, tức là rác. Việc còn lại là thu hồi chúng.

^code call-sweep (1 before, 2 after)

Toàn bộ logic nằm trong một hàm.

^code sweep

Tôi biết nhìn thì khá nhiều code và trò con trỏ, nhưng khi đi qua rồi thì không có gì phức tạp. Vòng lặp `while` bên ngoài duyệt linked list của toàn bộ object trong heap, kiểm tra mark bit của chúng. Nếu một object đã được đánh dấu (đen), ta để nguyên và đi tiếp. Nếu nó chưa được đánh dấu (trắng), ta bỏ liên kết nó khỏi danh sách và giải phóng bằng hàm `freeObject()` mà ta đã viết.

<img src="image/garbage-collection/unlink.png" alt="A recycle bin full of bits." />

Phần lớn code còn lại xử lý thực tế là gỡ một node khỏi singly linked list khá lằng nhằng. Ta phải luôn nhớ node trước đó để có thể bỏ liên kết con trỏ next của nó, và phải xử lý trường hợp cạnh khi đang giải phóng node đầu tiên. Còn lại thì rất đơn giản — xóa mọi node trong linked list mà không có bit được set.

Có một bổ sung nhỏ:

^code unmark (1 before, 1 after)

Sau khi `sweep()` hoàn tất, những object còn lại là các object đen “sống” với mark bit đang bật. Điều đó là đúng, nhưng khi chu kỳ collect *tiếp theo* bắt đầu, ta cần mọi object trở lại màu trắng. Vì vậy, bất cứ khi nào gặp một object đen, ta xóa bit ngay lúc này để sẵn sàng cho lần chạy tiếp theo.

### Weak reference & string pool

Chúng ta gần như đã xong phần collect. Còn một góc nhỏ của VM có yêu cầu hơi khác thường về bộ nhớ. Nhớ rằng khi thêm string vào clox, ta đã intern tất cả chúng. Điều đó có nghĩa VM giữ một hash table chứa con trỏ tới mọi string trong heap. VM dùng điều này để loại trùng string.

Trong giai đoạn mark, chúng ta cố tình *không* coi string table của VM là nguồn root. Nếu làm vậy, sẽ không <span name="intern">string</span> nào *bao giờ* được collect. String table sẽ phình to mãi và không bao giờ trả lại một byte nào cho hệ điều hành. Như vậy thì tệ.

<aside name="intern">

Đây có thể là vấn đề thực sự. Java không intern *tất cả* string, nhưng có intern string *literal*. Nó cũng cung cấp API để thêm string vào string table. Trong nhiều năm, dung lượng bảng đó là cố định, và string đã thêm thì không thể xóa. Nếu người dùng không cẩn thận với `String.intern()`, họ có thể hết bộ nhớ và crash.

Ruby từng gặp vấn đề tương tự trong nhiều năm với symbol — các giá trị giống string đã intern — không được garbage collected. Cuối cùng cả hai đều cho phép GC thu gom các string này.

</aside>

Đồng thời, nếu ta *cho phép* GC giải phóng string, thì string table của VM sẽ còn lại các con trỏ “rơi rớt” trỏ tới vùng nhớ đã được giải phóng. Điều đó còn tệ hơn.

String table là trường hợp đặc biệt và cần hỗ trợ đặc biệt. Cụ thể, nó cần một kiểu tham chiếu đặc biệt. Bảng này nên có thể tham chiếu tới một string, nhưng liên kết đó không được coi là root khi xác định reachability. Điều đó ngụ ý object được tham chiếu có thể bị giải phóng. Khi điều đó xảy ra, tham chiếu “rủ xuống” phải được dọn dẹp — kiểu như một con trỏ tự xóa một cách “ma thuật”. Tập hợp semantics này đủ phổ biến để có tên riêng: [**weak reference**][weak].

[weak]: https://en.wikipedia.org/wiki/Weak_reference

Chúng ta thực ra đã ngầm implement một nửa hành vi đặc thù của string table bằng việc *không* duyệt nó trong giai đoạn marking. Điều đó có nghĩa nó không ép string phải reachable. Phần còn lại là xóa mọi con trỏ rơi rớt tới các string đã bị giải phóng.

Để xóa các tham chiếu tới string unreachable, ta cần biết string nào *là* unreachable. Ta chỉ biết điều đó sau khi giai đoạn mark hoàn tất. Nhưng ta không thể chờ tới sau khi sweep xong vì lúc đó các object — và mark bit của chúng — đã không còn để kiểm tra. Vậy nên thời điểm đúng là ngay giữa hai giai đoạn marking và sweeping.

^code sweep-strings (1 before, 1 after)

Logic để xóa các string sắp bị xóa nằm trong một hàm mới ở module "table".

^code table-remove-white-h (2 before, 2 after)

Phần implement ở đây:

^code table-remove-white

Ta duyệt mọi entry trong bảng. Bảng intern string chỉ dùng key của mỗi entry — về cơ bản nó là một hash *set*, không phải hash *map*. Nếu mark bit của object string trong key chưa được set, thì đó là một object trắng, sắp bị quét bỏ. Ta xóa nó khỏi hash table trước, nhờ đó đảm bảo sẽ không còn con trỏ rơi rớt nào.


## Khi nào thì Collect

Giờ chúng ta đã có một mark-sweep garbage collector hoạt động đầy đủ. Khi bật stress testing flag, nó sẽ được gọi liên tục, và khi bật cả logging, ta có thể quan sát nó làm việc và thấy rõ nó thực sự thu hồi bộ nhớ. Nhưng khi tắt stress testing flag, nó sẽ không bao giờ chạy. Đã đến lúc quyết định khi nào collector nên được gọi trong quá trình chạy chương trình bình thường.

Theo như tôi thấy, câu hỏi này không được trả lời thỏa đáng trong các tài liệu. Khi garbage collector mới ra đời, máy tính chỉ có một lượng bộ nhớ nhỏ, cố định. Nhiều bài báo GC thời kỳ đầu giả định rằng bạn dành ra vài nghìn từ bộ nhớ — tức là phần lớn bộ nhớ — và gọi collector mỗi khi hết sạch. Đơn giản.

Máy hiện đại có hàng gigabyte RAM vật lý, được ẩn sau lớp trừu tượng bộ nhớ ảo còn lớn hơn của hệ điều hành, và được chia sẻ cho hàng loạt chương trình khác đang tranh nhau phần bộ nhớ của mình. Hệ điều hành sẽ cho phép chương trình của bạn yêu cầu bao nhiêu cũng được, rồi hoán đổi (page) vào/ra từ đĩa khi RAM vật lý đầy. Bạn sẽ không bao giờ thực sự “hết” bộ nhớ, chỉ là chương trình sẽ chạy chậm dần.

### Latency & throughput

Không còn hợp lý nếu chờ đến khi “bắt buộc” mới chạy GC, nên ta cần một chiến lược thời điểm tinh tế hơn. Để phân tích chính xác hơn, đã đến lúc giới thiệu hai con số cơ bản dùng để đo hiệu năng của một memory manager: *throughput* và *latency*.

Mọi ngôn ngữ managed đều phải trả giá hiệu năng so với việc giải phóng bộ nhớ thủ công do lập trình viên viết. Thời gian thực sự để giải phóng bộ nhớ là như nhau, nhưng GC phải tốn chu kỳ CPU để xác định *vùng nhớ nào* cần giải phóng. Đó là thời gian *không* dành cho việc chạy code của người dùng và làm công việc hữu ích. Trong implement của chúng ta, đó chính là toàn bộ giai đoạn mark. Mục tiêu của một garbage collector tinh vi là giảm thiểu overhead này.

Có hai chỉ số chính để hiểu rõ hơn chi phí đó:

*   **Throughput** là tỷ lệ tổng thời gian chạy code người dùng so với thời gian làm việc GC. Giả sử bạn chạy một chương trình clox trong 10 giây và dành 1 giây trong `collectGarbage()`. Điều đó nghĩa là throughput là 90% — 90% thời gian chạy chương trình và 10% cho overhead GC.

    Throughput là thước đo cơ bản nhất vì nó phản ánh tổng chi phí overhead của collection. Mọi yếu tố khác giữ nguyên, bạn muốn tối đa hóa throughput. Trước chương này, clox không có GC nên đạt <span name="hundred">100%</span> throughput. Khó mà vượt qua được. Tất nhiên, điều đó phải trả giá bằng nguy cơ hết bộ nhớ và crash nếu chương trình người dùng chạy đủ lâu. Bạn có thể coi mục tiêu của GC là sửa “lỗi” đó trong khi hy sinh ít throughput nhất có thể.

<aside name="hundred">

Thực ra không *hoàn toàn* 100%. Nó vẫn phải đưa các object được cấp phát vào linked list, nên cũng có chút overhead nhỏ để set các con trỏ đó.

</aside>

*   **Latency** là khoảng thời gian *liên tục* dài nhất mà chương trình người dùng bị dừng hoàn toàn để chạy garbage collection. Đây là thước đo mức độ “gián đoạn” của collector. Latency là một chỉ số hoàn toàn khác với throughput.

    Hãy so sánh hai lần chạy chương trình clox, mỗi lần đều mất 10 giây. Lần đầu, GC chạy một lần và mất trọn 1 giây trong `collectGarbage()` cho một đợt collect lớn. Lần thứ hai, GC được gọi 5 lần, mỗi lần 1/5 giây. Tổng thời gian collect vẫn là 1 giây, nên throughput ở cả hai trường hợp đều là 90%. Nhưng ở lần thứ hai, latency chỉ là 1/5 giây, thấp hơn 5 lần so với lần đầu.

<span name="latency"></span>

<img src="image/garbage-collection/latency-throughput.png" alt="Một thanh biểu diễn thời gian execute với các phần cho chạy code người dùng và chạy GC. Phần GC lớn nhất là latency. Tổng kích thước các phần code người dùng là throughput." />

<aside name="latency">

Thanh này biểu diễn quá trình execute chương trình, chia thành thời gian chạy code người dùng và thời gian chạy GC. Kích thước của phần thời gian chạy GC lớn nhất là latency. Tổng kích thước các phần code người dùng cộng lại là throughput.

</aside>

Nếu bạn thích ví von, hãy tưởng tượng chương trình của bạn là một tiệm bánh bán bánh mì mới nướng cho khách. Throughput là tổng số bánh mì nóng giòn bạn phục vụ được trong một ngày. Latency là thời gian mà vị khách xui nhất phải chờ trong hàng trước khi được phục vụ.

<span name="dishwasher">Chạy</span> garbage collector giống như tạm thời đóng cửa tiệm để rửa bát đĩa, phân loại đồ bẩn và đồ sạch, rồi rửa sạch đồ đã dùng. Trong ví dụ này, ta không có nhân viên rửa bát riêng, nên khi việc này diễn ra, không có bánh nào được nướng. Thợ làm bánh đang đi rửa bát.

<aside name="dishwasher">

Nếu mỗi người là một thread, thì tối ưu hiển nhiên là có thread riêng chạy garbage collection, tạo thành **concurrent garbage collector**. Nói cách khác, thuê vài người rửa bát trong khi người khác vẫn nướng bánh. Đây là cách các GC rất tinh vi hoạt động vì nó cho phép thợ làm bánh — các worker thread — tiếp tục chạy code người dùng với ít gián đoạn.

Tuy nhiên, cần có sự phối hợp. Bạn không muốn người rửa bát giật cái bát khỏi tay thợ làm bánh! Sự phối hợp này thêm overhead và nhiều phức tạp. Concurrent collector nhanh, nhưng khó implement đúng.

<img src="image/garbage-collection/baguette.png" class="above" alt="Un baguette." />

</aside>

Bán ít bánh hơn mỗi ngày là điều tệ, và bắt một khách nào đó phải ngồi chờ trong khi bạn rửa hết đống bát đĩa cũng tệ. Mục tiêu là tối đa throughput và tối thiểu latency, nhưng không có bữa trưa miễn phí, ngay cả trong tiệm bánh. Garbage collector đưa ra những đánh đổi khác nhau giữa lượng throughput hy sinh và mức latency chấp nhận.

Khả năng điều chỉnh những đánh đổi này là hữu ích vì các chương trình người dùng khác nhau có nhu cầu khác nhau. Một batch job chạy qua đêm để tạo báo cáo từ một terabyte dữ liệu chỉ cần hoàn thành càng nhiều việc càng nhanh càng tốt. Throughput là tối thượng. Trong khi đó, một ứng dụng chạy trên smartphone của người dùng cần luôn phản hồi ngay lập tức với thao tác để việc kéo thả trên màn hình cảm giác mượt <span name="butter">buttery</span>. Ứng dụng không thể “đóng băng” vài giây trong khi GC lục lọi heap.

<aside name="butter">

Rõ ràng là ví dụ tiệm bánh đang ám ảnh tôi.

</aside>

Là tác giả garbage collector, bạn kiểm soát một phần sự đánh đổi giữa throughput và latency bằng cách chọn thuật toán collect. Nhưng ngay cả trong một thuật toán duy nhất, ta vẫn có nhiều quyền kiểm soát *tần suất* collector chạy.

Collector của chúng ta là <span name="incremental">**stop-the-world GC**</span>, nghĩa là chương trình người dùng bị tạm dừng cho đến khi toàn bộ quá trình garbage collection hoàn tất. Nếu ta chờ lâu mới chạy collector, thì số lượng object chết sẽ tích tụ nhiều. Điều đó dẫn đến một lần dừng rất lâu khi collector chạy, và vì thế latency cao. Vậy nên, rõ ràng là ta muốn chạy collector thường xuyên hơn.

<aside name="incremental">

Ngược lại, **incremental garbage collector** có thể collect một chút, rồi chạy code người dùng, rồi collect thêm chút nữa, và cứ thế.

</aside>


Nhưng mỗi lần collector chạy, nó sẽ tốn thời gian để duyệt qua các live object. Việc này thực ra không *làm* được gì hữu ích (ngoài việc đảm bảo chúng không bị xóa nhầm). Thời gian duyệt live object là thời gian không giải phóng bộ nhớ, và cũng là thời gian không chạy code của người dùng. Nếu bạn chạy GC *quá* thường xuyên, chương trình của người dùng thậm chí không có đủ thời gian để tạo ra rác mới cho VM thu gom. VM sẽ dành toàn bộ thời gian để ám ảnh lặp đi lặp lại cùng một tập live object, và throughput sẽ giảm. Vậy nên, rõ ràng là ta muốn chạy collector thật *ít*.

Thực tế, ta muốn một điểm ở giữa, và tần suất collector chạy là một trong những “núm vặn” chính để tinh chỉnh sự đánh đổi giữa latency và throughput.

### Self-adjusting heap

Ta muốn GC chạy đủ thường xuyên để giảm latency, nhưng cũng đủ thưa để giữ throughput ở mức tốt. Nhưng làm sao tìm được điểm cân bằng khi ta không biết chương trình của người dùng cần bao nhiêu bộ nhớ và tần suất cấp phát ra sao? Ta có thể đẩy vấn đề này cho người dùng và bắt họ tự chọn bằng cách cung cấp các tham số tinh chỉnh GC. Nhiều VM làm vậy. Nhưng nếu chính chúng ta — tác giả GC — còn không biết tinh chỉnh thế nào cho tốt, thì khả năng cao là đa số người dùng cũng không biết. Họ xứng đáng có một hành vi mặc định hợp lý.

Thành thật mà nói, đây không phải lĩnh vực tôi giỏi. Tôi đã nói chuyện với nhiều “hacker” GC chuyên nghiệp — đây là thứ bạn có thể xây dựng cả sự nghiệp — và đọc nhiều tài liệu, nhưng tất cả câu trả lời tôi nhận được đều… mơ hồ. Chiến lược tôi chọn là một cách phổ biến, khá đơn giản, và (hy vọng là) đủ tốt cho hầu hết trường hợp.

Ý tưởng là tần suất collector sẽ tự động điều chỉnh dựa trên kích thước live của heap. Ta theo dõi tổng số byte bộ nhớ managed mà VM đã cấp phát. Khi nó vượt quá một ngưỡng nào đó, ta kích hoạt GC. Sau đó, ta ghi nhận số byte bộ nhớ còn lại — tức là số byte *không* được giải phóng. Rồi ta điều chỉnh ngưỡng lên một giá trị lớn hơn con số đó.

Kết quả là khi lượng live memory tăng, ta collect ít thường xuyên hơn để tránh hy sinh throughput do phải duyệt lại đống live object ngày càng lớn. Khi lượng live memory giảm, ta collect thường xuyên hơn để không mất quá nhiều latency vì chờ quá lâu.

Việc implement cần thêm hai trường bookkeeping mới trong VM.

^code vm-fields (1 before, 1 after)

Trường đầu tiên là tổng số byte bộ nhớ managed mà VM đã cấp phát. Trường thứ hai là ngưỡng kích hoạt lần collect tiếp theo. Ta khởi tạo chúng khi VM khởi động.

^code init-gc-fields (1 before, 2 after)

Ngưỡng ban đầu ở đây là <span name="lab">tùy ý</span>. Nó giống như dung lượng khởi tạo mà ta chọn cho các dynamic array trước đây. Mục tiêu là không kích hoạt vài lần GC đầu *quá* nhanh nhưng cũng không để chờ quá lâu. Nếu ta có một số chương trình Lox thực tế, ta có thể profile để tinh chỉnh. Nhưng vì hiện tại chỉ có các chương trình “đồ chơi”, tôi chọn đại một con số.

<aside name="lab">

Một thách thức khi học về garbage collector là *rất* khó tìm ra best practice trong môi trường lab tách biệt. Bạn sẽ không thấy collector thực sự hoạt động thế nào trừ khi chạy nó trên những chương trình lớn, phức tạp, lộn xộn ngoài đời thực — đúng kiểu mà nó được thiết kế để xử lý. Giống như tinh chỉnh xe đua rally — bạn phải mang nó ra đường đua.

</aside>

Mỗi lần ta cấp phát hoặc giải phóng bộ nhớ, ta điều chỉnh bộ đếm theo delta đó.

^code updated-bytes-allocated (1 before, 1 after)

Khi tổng số vượt ngưỡng, ta chạy collector.

^code collect-on-next (2 before, 1 after)

Giờ thì cuối cùng, garbage collector của chúng ta thực sự làm việc khi người dùng chạy chương trình mà không bật hidden diagnostic flag. Giai đoạn sweep giải phóng object bằng cách gọi `reallocate()`, điều này làm giảm giá trị `bytesAllocated`, nên sau khi collect xong, ta biết còn lại bao nhiêu byte live. Ta điều chỉnh ngưỡng cho lần GC tiếp theo dựa trên con số đó.

^code update-next-gc (1 before, 2 after)

Ngưỡng này là một bội số của kích thước heap. Bằng cách này, khi lượng bộ nhớ chương trình dùng tăng, ngưỡng sẽ lùi xa hơn để giới hạn tổng thời gian duyệt lại tập live lớn hơn. Giống như các con số khác trong chương này, hệ số nhân này về cơ bản là tùy ý.

^code heap-grow-factor (1 before, 2 after)

Bạn sẽ muốn tinh chỉnh nó trong implement của mình khi có một số chương trình thực tế để benchmark. Còn bây giờ, ít nhất ta có thể log một số thống kê hiện có. Ta ghi nhận kích thước heap trước khi collect.

^code log-before-size (1 before, 1 after)

Và in kết quả ở cuối.

^code log-collected-amount (1 before, 1 after)

Bằng cách này, ta có thể thấy garbage collector đã làm được bao nhiêu việc trong lúc chạy.

## Lỗi trong Garbage Collection

Về lý thuyết, đến đây là chúng ta đã xong. Chúng ta có một GC. Nó chạy định kỳ, thu gom những gì có thể, và để lại phần còn lại. Nếu đây là một cuốn giáo trình thông thường, ta sẽ phủi bụi khỏi tay và tận hưởng ánh hào quang của tòa kiến trúc cẩm thạch hoàn hảo mà mình vừa tạo ra.

Nhưng tôi muốn dạy bạn không chỉ lý thuyết về ngôn ngữ lập trình mà cả thực tế đôi khi đầy đau đớn. Tôi sẽ lật một khúc gỗ mục và cho bạn thấy những con bọ gớm ghiếc sống bên dưới, và lỗi trong garbage collector thực sự là một trong những loài “không xương” kinh khủng nhất.

Nhiệm vụ của collector là giải phóng các object chết và giữ lại các object sống. Sai lầm có thể xảy ra dễ dàng ở cả hai hướng. Nếu VM không giải phóng các object không cần thiết, nó sẽ rò rỉ bộ nhớ từ từ. Nếu nó giải phóng một object vẫn đang được dùng, chương trình của người dùng có thể truy cập vào vùng nhớ không hợp lệ. Những lỗi này thường không gây crash ngay lập tức, khiến việc lần ngược lại để tìm nguyên nhân trở nên khó khăn.

Điều này càng khó hơn vì ta không biết khi nào collector sẽ chạy. Bất kỳ lời gọi nào cuối cùng cũng dẫn đến việc cấp phát bộ nhớ đều là một điểm trong VM nơi collection có thể xảy ra. Nó giống như trò “ghế nhạc”. Bất cứ lúc nào, GC cũng có thể dừng nhạc. Mỗi object được cấp phát trên heap mà ta muốn giữ lại cần phải nhanh chóng “tìm được ghế” — được đánh dấu là root hoặc được lưu dưới dạng tham chiếu trong một object khác — trước khi giai đoạn sweep đến và đá nó ra khỏi cuộc chơi.

Làm sao VM có thể dùng một object sau này — một object mà GC không thấy? Làm sao VM tìm được nó? Câu trả lời phổ biến nhất là thông qua một con trỏ được lưu trong biến local trên C stack. GC sẽ duyệt qua value stack và CallFrame stack của *VM*, nhưng C stack thì lại <span name="c">ẩn</span> với nó.

<aside name="c">

GC của chúng ta không thể tìm địa chỉ trong C stack, nhưng nhiều GC khác thì có thể. Conservative garbage collector sẽ quét toàn bộ bộ nhớ, bao gồm cả native stack. Loại nổi tiếng nhất là [**Boehm–Demers–Weiser garbage collector**][boehm], thường được gọi ngắn gọn là “Boehm collector”. (Con đường ngắn nhất để nổi tiếng trong ngành khoa học máy tính là có họ đứng đầu bảng chữ cái để tên bạn xuất hiện đầu tiên trong danh sách sắp xếp theo tên.)

[boehm]: https://en.wikipedia.org/wiki/Boehm_garbage_collector

Nhiều precise GC cũng duyệt C stack. Ngay cả những GC đó cũng phải cẩn thận với các con trỏ tới object sống chỉ tồn tại trong *thanh ghi CPU*.

</aside>

Trong các chương trước, chúng ta đã viết những đoạn code tưởng như vô nghĩa: đẩy một object lên value stack của VM, làm một chút việc, rồi lại pop nó xuống. Hầu hết thời gian, tôi nói rằng điều này là để phục vụ GC. Giờ bạn đã thấy lý do. Đoạn code giữa push và pop có thể cấp phát bộ nhớ và do đó kích hoạt GC. Ta phải đảm bảo object nằm trên value stack để giai đoạn mark của collector tìm thấy và giữ nó sống.

Tôi đã viết toàn bộ implement của clox trước khi chia nó thành các chương và viết phần giải thích, nên tôi có nhiều thời gian để tìm ra những góc khuất này và loại bỏ hầu hết các lỗi. Phần code stress testing mà ta thêm vào đầu chương này và một bộ test khá tốt đã giúp ích rất nhiều.

Nhưng tôi chỉ sửa *hầu hết* chúng. Tôi cố tình để lại vài lỗi vì muốn bạn cảm nhận được việc gặp những lỗi này “ngoài đời” như thế nào. Nếu bạn bật stress test flag và chạy vài chương trình Lox nhỏ, có thể bạn sẽ bắt gặp một vài lỗi. Hãy thử xem *bạn có thể tự sửa được cái nào không*.

### Thêm vào constant table

Bạn rất có thể sẽ gặp lỗi đầu tiên này. Constant table mà mỗi chunk sở hữu là một dynamic array. Khi compiler thêm một constant mới vào bảng của function hiện tại, mảng này có thể cần mở rộng. Bản thân constant đó cũng có thể là một object được cấp phát trên heap như string hoặc function lồng nhau.

Object mới được thêm vào constant table sẽ được truyền vào `addConstant()`. Tại thời điểm đó, object này chỉ tồn tại trong tham số của hàm trên C stack. Hàm này sẽ append object vào constant table. Nếu bảng không đủ dung lượng và cần mở rộng, nó sẽ gọi `reallocate()`. Điều này sẽ kích hoạt GC, và GC sẽ không đánh dấu object constant mới, dẫn đến việc quét và xóa nó ngay trước khi ta kịp thêm vào bảng. Crash.

Cách sửa, như bạn đã thấy ở những chỗ khác, là push constant đó lên stack tạm thời.

^code add-constant-push (1 before, 1 after)

Khi constant table đã chứa object, ta pop nó khỏi stack.

^code add-constant-pop (1 before, 1 after)

Khi GC đánh dấu root, nó sẽ duyệt chuỗi compiler và đánh dấu từng function của chúng, nên constant mới giờ đã reachable. Ta cần thêm include để gọi vào VM từ module "chunk".

^code chunk-include-vm (1 before, 2 after)

### Interning string

Đây là một lỗi tương tự. Tất cả string trong clox đều được intern, nên mỗi khi tạo string mới, ta cũng thêm nó vào intern table. Bạn có thể đoán được chuyện gì xảy ra. Vì string này hoàn toàn mới, nó chưa reachable ở đâu cả. Và việc resize string pool có thể kích hoạt collection. Một lần nữa, ta sẽ stash string này lên stack trước.

^code push-string (2 before, 1 after)

Rồi pop nó xuống khi nó đã an toàn trong bảng.

^code pop-string (1 before, 2 after)

Điều này đảm bảo string an toàn trong khi bảng đang được resize. Khi đã vượt qua bước này, `allocateString()` sẽ trả nó về cho caller, và caller sẽ chịu trách nhiệm đảm bảo string vẫn reachable trước khi lần cấp phát heap tiếp theo diễn ra.

### Nối chuỗi (Concatenating strings)

Một ví dụ cuối cùng: Trong interpreter, lệnh `OP_ADD` có thể được dùng để nối hai chuỗi. Giống như với số, nó pop hai toán hạng khỏi stack, tính toán kết quả, rồi push giá trị mới đó trở lại stack. Với số thì điều này hoàn toàn an toàn.

Nhưng nối hai chuỗi lại đòi hỏi phải cấp phát một mảng ký tự mới trên heap, và điều này có thể kích hoạt GC. Vì ở thời điểm đó ta đã pop các chuỗi toán hạng ra khỏi stack, chúng có thể bị bỏ sót trong giai đoạn mark và bị quét mất. Thay vì pop chúng ra khỏi stack ngay lập tức, ta sẽ peek chúng.

^code concatenate-peek (1 before, 2 after)

Bằng cách này, chúng vẫn còn nằm trên stack khi ta tạo chuỗi kết quả. Khi xong, ta có thể pop chúng ra một cách an toàn và thay thế bằng kết quả.

^code concatenate-pop (1 before, 1 after)

Những lỗi này khá dễ xử lý, nhất là vì tôi *chỉ cho* bạn chỗ cần sửa. Trong thực tế, *tìm ra* chúng mới là phần khó. Tất cả những gì bạn thấy là một object *đáng lẽ* phải tồn tại nhưng lại không. Nó không giống các lỗi khác, nơi bạn tìm đoạn code *gây ra* vấn đề. Ở đây, bạn tìm *sự thiếu vắng* của đoạn code vốn có nhiệm vụ *ngăn chặn* vấn đề, và đó là một cuộc tìm kiếm khó hơn nhiều.

Nhưng ít nhất là bây giờ, bạn có thể yên tâm. Theo như tôi biết, chúng ta đã tìm ra tất cả lỗi liên quan đến collection trong clox, và giờ ta có một mark-sweep garbage collector hoạt động tốt, ổn định và tự điều chỉnh.

<div class="challenges">

## Thử thách

1.  Struct header `Obj` ở đầu mỗi object giờ có ba trường: `type`, `isMarked`, và `next`. Chúng chiếm bao nhiêu bộ nhớ (trên máy của bạn)? Bạn có thể nghĩ ra cách nào gọn hơn không? Liệu có chi phí runtime khi làm vậy không?

2.  Khi giai đoạn sweep duyệt qua một object sống, nó sẽ xóa trường `isMarked` để chuẩn bị cho chu kỳ collection tiếp theo. Bạn có thể nghĩ ra cách nào hiệu quả hơn không?

3.  Mark-sweep chỉ là một trong nhiều thuật toán garbage collection hiện có. Hãy khám phá bằng cách thay thế hoặc bổ sung collector hiện tại bằng một loại khác. Một số ứng viên đáng cân nhắc: reference counting, thuật toán Cheney, hoặc thuật toán mark-compact của Lisp 2.

</div>

<div class="design-note">

## Ghi chú thiết kế: Generational Collectors

Một collector sẽ mất throughput nếu tốn nhiều thời gian duyệt lại các object vẫn còn sống. Nhưng nếu tránh collect và để tích tụ một đống rác lớn, latency sẽ tăng. Giá như có cách nào để biết object nào có khả năng sống lâu và object nào thì không. Khi đó, GC có thể tránh duyệt lại các object sống lâu quá thường xuyên và dọn dẹp các object ngắn hạn thường xuyên hơn.

Thực tế là có. Nhiều năm trước, các nhà nghiên cứu GC đã thu thập số liệu về vòng đời của object trong các chương trình thực tế. Họ theo dõi từng object khi nó được cấp phát, và khi nó không còn cần thiết nữa, rồi vẽ biểu đồ về thời gian sống của chúng.

Họ phát hiện ra một điều gọi là **generational hypothesis**, hay một cách nói kém tế nhị hơn là **infant mortality**. Quan sát của họ là hầu hết object đều sống rất ngắn, nhưng một khi chúng sống qua một độ tuổi nhất định, chúng thường tồn tại khá lâu. Object *càng* sống lâu, khả năng nó *tiếp tục* sống càng cao. Quan sát này rất hữu ích vì nó cho phép phân loại object thành nhóm cần collect thường xuyên và nhóm không cần.

Họ đã thiết kế một kỹ thuật gọi là **generational garbage collection**. Cách hoạt động như sau: Mỗi khi một object mới được cấp phát, nó sẽ vào một vùng đặc biệt, tương đối nhỏ của heap gọi là "nursery". Vì object thường “chết trẻ”, garbage collector sẽ được gọi <span name="nursery">thường xuyên</span> chỉ trên các object trong vùng này.

<aside name="nursery">

Nursery thường được quản lý bằng copying collector, vốn nhanh hơn mark-sweep collector trong việc cấp phát và giải phóng object.

</aside>

Mỗi lần GC chạy trên nursery được gọi là một “generation”. Bất kỳ object nào không còn cần thiết sẽ bị giải phóng. Những object sống sót sẽ được coi là già thêm một thế hệ, và GC sẽ theo dõi điều này cho từng object. Nếu một object sống sót qua một số thế hệ nhất định — thường chỉ một lần collect — nó sẽ được *tenure*. Lúc này, nó được sao chép ra khỏi nursery sang một vùng heap lớn hơn dành cho các object sống lâu. Garbage collector cũng sẽ chạy trên vùng này, nhưng ít thường xuyên hơn vì khả năng cao là hầu hết các object ở đây vẫn còn sống.

Generational collector là sự kết hợp tuyệt vời giữa dữ liệu thực nghiệm — quan sát rằng vòng đời object *không* phân bố đều — và thiết kế thuật toán thông minh tận dụng thực tế đó. Chúng cũng khá đơn giản về mặt khái niệm. Bạn có thể hình dung nó như hai GC được tinh chỉnh riêng biệt và một chính sách khá đơn giản để di chuyển object từ vùng này sang vùng khác.

</div>