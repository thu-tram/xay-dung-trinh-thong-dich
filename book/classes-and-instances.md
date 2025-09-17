> Quan tâm quá nhiều đến đồ vật có thể hủy hoại bạn. Nhưng — nếu bạn quan tâm đến một thứ đủ nhiều, nó sẽ có một “cuộc sống” riêng, đúng không? Và chẳng phải ý nghĩa của mọi thứ — những thứ đẹp đẽ — là chúng kết nối bạn với một vẻ đẹp lớn hơn sao?
>
> <cite>Donna Tartt, <em>The Goldfinch</em></cite>

Phần cuối cùng còn lại để cài đặt trong clox là lập trình hướng đối tượng. <span name="oop">OOP</span> là một gói các tính năng đan xen nhau: class, instance, field, method, initializer, và inheritance. Khi dùng Java — một ngôn ngữ bậc cao hơn — chúng ta gói gọn tất cả trong hai chương. Giờ đây, khi lập trình bằng C, cảm giác như đang dựng mô hình tháp Eiffel bằng… tăm xỉa răng, chúng ta sẽ dành hẳn ba chương để bao quát cùng một phạm vi. Điều này cho phép chúng ta thong thả hơn khi triển khai. Sau những chương “nặng đô” như [closures](closures.html) và [garbage collector](garbage-collection.html), bạn xứng đáng được nghỉ ngơi. Thực tế, từ đây trở đi, cuốn sách sẽ dễ hơn nhiều.

<aside name="oop">

Những người có quan điểm mạnh mẽ về lập trình hướng đối tượng — tức là “ai cũng vậy” — thường cho rằng OOP đồng nghĩa với một danh sách rất cụ thể các tính năng ngôn ngữ. Nhưng thực ra, đây là một không gian rộng để khám phá, và mỗi ngôn ngữ lại có “nguyên liệu” và “công thức” riêng.

Self có object nhưng không có class. CLOS có method nhưng không gắn chúng với class cụ thể. C++ ban đầu không có runtime polymorphism — không có virtual method. Python có multiple inheritance, nhưng Java thì không. Ruby gắn method vào class, nhưng bạn cũng có thể định nghĩa method cho một object đơn lẻ.

</aside>

Trong chương này, chúng ta sẽ triển khai ba tính năng đầu tiên: class, instance, và field. Đây là phần “có trạng thái” của hướng đối tượng. Sau đó, trong hai chương tiếp theo, chúng ta sẽ gắn hành vi và khả năng tái sử dụng code vào các object này.

## Class Objects

Trong một ngôn ngữ hướng đối tượng dựa trên class, mọi thứ bắt đầu từ class. Chúng định nghĩa loại object nào tồn tại trong chương trình và là “nhà máy” tạo ra các instance mới. Đi từ dưới lên, chúng ta sẽ bắt đầu với cách biểu diễn chúng ở runtime, rồi kết nối nó vào ngôn ngữ.

Đến giờ, chúng ta đã quen với quy trình thêm một loại object mới vào VM. Ta bắt đầu với một struct.

^code obj-class (1 before, 2 after)

Sau phần header Obj, chúng ta lưu tên của class. Điều này không thực sự cần thiết cho chương trình của người dùng, nhưng nó cho phép chúng ta hiển thị tên ở runtime, ví dụ như trong stack trace.

Loại mới này cần một case tương ứng trong enum ObjType.

^code obj-type-class (1 before, 1 after)

Và loại này cũng cần một cặp macro tương ứng. Đầu tiên, để kiểm tra kiểu của một object:

^code is-class (2 before, 1 after)

Và sau đó là để cast một Value thành con trỏ ObjClass:

^code as-class (2 before, 1 after)

VM tạo class object mới bằng hàm sau:

^code new-class-h (2 before, 1 after)

Phần cài đặt nằm ở đây:

^code new-class

Hầu hết chỉ là code mẫu. Hàm nhận tên class dưới dạng string và lưu lại. Mỗi khi người dùng khai báo một class mới, VM sẽ tạo một ObjClass struct mới để biểu diễn nó.

<aside name="klass">

<img src="image/classes-and-instances/klass.png" alt="'Klass' in a zany kidz font."/>

Tôi đặt tên biến là “klass” không chỉ để tạo cảm giác “Kidz Korner” vui nhộn cho VM. Nó còn giúp việc biên dịch clox bằng C++ dễ dàng hơn, vì “class” là từ khóa trong C++.

</aside>

Khi VM không còn cần class nữa, nó sẽ giải phóng như sau:

^code free-class (1 before, 1 after)

<aside name="braces">

Dấu ngoặc ở đây hiện tại chưa cần thiết, nhưng sẽ hữu ích trong chương sau khi chúng ta thêm code vào nhánh switch case này.

</aside>

Giờ chúng ta đã có bộ quản lý bộ nhớ, nên cũng cần hỗ trợ việc “tracing” qua các class object.

^code blacken-class (1 before, 1 after)

Khi GC gặp một class object, nó sẽ đánh dấu tên class để giữ cho string đó không bị giải phóng.

Thao tác cuối cùng mà VM có thể thực hiện trên một class là in ra nó.

^code print-class (1 before, 1 after)

Một class chỉ đơn giản là in ra tên của chính nó.

## Class Declarations

Khi đã có cách biểu diễn ở runtime, chúng ta sẵn sàng thêm hỗ trợ cho class vào ngôn ngữ. Tiếp theo, ta chuyển sang parser.

^code match-class (1 before, 1 after)

Khai báo class là một statement, và parser nhận diện nó bằng từ khóa `class` ở đầu. Phần compile còn lại diễn ra ở đây:

^code class-declaration

Ngay sau từ khóa `class` là tên class. Chúng ta lấy identifier đó và thêm vào constant table của hàm bao quanh dưới dạng string. Như bạn vừa thấy, in ra một class sẽ hiển thị tên của nó, nên compiler cần lưu string tên này ở nơi mà runtime có thể tìm thấy. Constant table chính là cách để làm điều đó.

Tên <span name="variable">class</span> này cũng được dùng để gán class object vào một biến cùng tên. Vì vậy, ngay sau khi đọc token tên class, ta khai báo một biến với identifier đó.

<aside name="variable">

Chúng ta hoàn toàn có thể thiết kế để khai báo class là *biểu thức* thay vì statement — về bản chất, chúng cũng giống như một literal tạo ra một giá trị. Khi đó, người dùng sẽ phải tự gán class vào một biến, ví dụ:

```lox
var Pie = class {}
```

Giống như lambda function nhưng dành cho class. Tuy nhiên, vì chúng ta thường muốn class có tên, nên hợp lý hơn khi coi chúng là một dạng khai báo.

</aside>

Tiếp theo, chúng ta sinh ra một instruction mới để thực sự tạo class object tại runtime. Instruction này nhận chỉ số trong constant table của tên class làm operand.

Sau đó, nhưng trước khi compile phần thân class, chúng ta define biến cho tên class. *Khai báo* biến sẽ thêm nó vào scope, nhưng nhớ lại từ [một chương trước](local-variables.html#another-scope-edge-case) rằng ta không thể *dùng* biến cho đến khi nó được *define*. Với class, ta define biến trước phần thân. Cách này cho phép người dùng tham chiếu đến class chứa nó ngay bên trong các method của chính nó. Điều này hữu ích cho những thứ như factory method tạo ra instance mới của class.

Cuối cùng, chúng ta compile phần thân. Hiện tại chưa có method, nên nó chỉ đơn giản là một cặp dấu ngoặc nhọn rỗng. Lox không yêu cầu khai báo field trong class, nên ta tạm xong phần thân — và cả parser — ở đây.

Compiler đang sinh ra một instruction mới, vậy hãy định nghĩa nó.

^code class-op (1 before, 1 after)

Và thêm nó vào disassembler:

^code disassemble-class (2 before, 1 after)

Với một tính năng trông có vẻ “to lớn” như vậy, phần hỗ trợ trong interpreter lại rất tối giản.

^code interpret-class (2 before, 1 after)

Chúng ta load string tên class từ constant table và truyền nó vào `newClass()`. Hàm này tạo một class object mới với tên đã cho. Ta push object đó lên stack và xong. Nếu class được gán vào một biến global, thì lời gọi `defineVariable()` của compiler sẽ sinh code để lưu object đó từ stack vào bảng biến global. Nếu không, nó đã nằm đúng vị trí trên stack để dùng cho một biến <span name="local">local</span> mới.

<aside name="local">

Class “local” — class được khai báo bên trong thân hàm hoặc block — là một khái niệm khá lạ. Nhiều ngôn ngữ không cho phép điều này. Nhưng vì Lox là một ngôn ngữ scripting kiểu dynamic, nó xử lý phần top-level của chương trình và thân hàm/block theo cùng một cách. Class chỉ là một dạng khai báo khác, và vì bạn có thể khai báo biến và hàm bên trong block, bạn cũng có thể khai báo class ở đó.

</aside>

Vậy là xong, VM của chúng ta giờ đã hỗ trợ class. Bạn có thể chạy:

```lox
class Brioche {}
print Brioche;
```

Tiếc là hiện tại in ra gần như là *tất cả* những gì bạn có thể làm với class, nên bước tiếp theo là làm cho chúng hữu ích hơn.

## Instance của Class

Class có hai mục đích chính trong một ngôn ngữ:

*   **Chúng là cách để tạo instance mới.** Đôi khi điều này liên quan đến từ khóa `new`, đôi khi là một lời gọi method trên class object, nhưng thường bạn sẽ nhắc đến class bằng tên *nào đó* để tạo instance mới.

*   **Chúng chứa method.** Đây là nơi định nghĩa cách tất cả instance của class hoạt động.

Chúng ta sẽ chưa bàn đến method cho đến chương sau, nên bây giờ chỉ tập trung vào phần đầu tiên. Trước khi class có thể tạo instance, chúng ta cần một cách biểu diễn chúng.

^code obj-instance (1 before, 2 after)

Instance biết class của mình — mỗi instance có một con trỏ tới class mà nó là instance của. Chúng ta sẽ chưa dùng nhiều trong chương này, nhưng nó sẽ trở nên quan trọng khi thêm method.

Quan trọng hơn trong chương này là cách instance lưu trữ trạng thái. Lox cho phép người dùng tự do thêm field vào instance tại runtime. Điều này có nghĩa là ta cần một cơ chế lưu trữ có thể mở rộng. Ta có thể dùng mảng động, nhưng ta cũng muốn tra cứu field theo tên càng nhanh càng tốt. Có một cấu trúc dữ liệu hoàn hảo cho việc truy cập nhanh một tập giá trị theo tên và — tiện hơn nữa — chúng ta đã cài đặt nó rồi. Mỗi instance lưu field của mình bằng một hash table.

<aside name="fields">

Khả năng tự do thêm field vào object tại runtime là một khác biệt lớn về mặt thực tiễn giữa hầu hết ngôn ngữ dynamic và static. Ngôn ngữ static thường yêu cầu field phải được khai báo rõ ràng. Cách này giúp compiler biết chính xác mỗi instance có những field nào. Nó có thể dùng thông tin đó để xác định chính xác lượng bộ nhớ cần cho mỗi instance và offset trong bộ nhớ nơi mỗi field được lưu.

Trong Lox và các ngôn ngữ dynamic khác, việc truy cập field thường là một thao tác tra cứu hash table. Thời gian hằng số, nhưng vẫn khá “nặng”. Trong một ngôn ngữ như C++, truy cập field nhanh như việc cộng một offset hằng số vào con trỏ.

</aside>

Chúng ta chỉ cần thêm một include, và xong.

^code object-include-table (1 before, 1 after)

Struct mới này có một object type mới.

^code obj-type-instance (1 before, 1 after)

Tôi muốn chậm lại một chút ở đây vì khái niệm “type” trong *ngôn ngữ* Lox và khái niệm “type” trong *cài đặt* VM có thể gây nhầm lẫn. Trong code C tạo nên clox, có nhiều loại Obj khác nhau — ObjString, ObjClosure, v.v. Mỗi loại có cách biểu diễn nội bộ và ngữ nghĩa riêng.

Trong *ngôn ngữ* Lox, người dùng có thể định nghĩa class của riêng mình — ví dụ Cake và Pie — rồi tạo instance của các class đó. Từ góc nhìn của người dùng, một instance của Cake là một loại object khác với một instance của Pie. Nhưng từ góc nhìn của VM, mỗi class mà người dùng định nghĩa chỉ đơn giản là một giá trị khác của kiểu ObjClass. Tương tự, mỗi instance trong chương trình của người dùng, bất kể thuộc class nào, đều là một ObjInstance. Một loại object của VM bao quát instance của mọi class. Hai “thế giới” này ánh xạ với nhau như sau:

<img src="image/classes-and-instances/lox-clox.png" alt="Một tập các khai báo class và instance, và cách biểu diễn runtime mà mỗi cái ánh xạ tới."/>

Rõ chưa? OK, quay lại phần cài đặt. Chúng ta cũng có các macro quen thuộc.

^code is-instance (1 before, 1 after)

Và:

^code as-instance (1 before, 1 after)

Vì field được thêm sau khi instance được tạo, hàm “constructor” chỉ cần biết class.

^code new-instance-h (1 before, 1 after)


Chúng ta cài đặt hàm này ở đây:

^code new-instance

Ta lưu một tham chiếu tới class của instance. Sau đó khởi tạo bảng field thành một hash table rỗng. Một “em bé” object mới ra đời!

Ở “đầu bên kia” buồn hơn của vòng đời instance, nó sẽ bị giải phóng.

^code free-instance (3 before, 1 after)

Instance sở hữu bảng field của nó, nên khi giải phóng instance, ta cũng giải phóng bảng này. Ta không giải phóng trực tiếp các entry *bên trong* bảng, vì có thể vẫn còn tham chiếu khác tới các object đó. Garbage collector sẽ lo phần này cho chúng ta. Ở đây, ta chỉ giải phóng mảng entry của chính bảng.

Nói đến garbage collector, nó cũng cần hỗ trợ việc “tracing” qua các instance.

^code blacken-instance (3 before, 1 after)

Nếu instance còn sống, ta cần giữ lại class của nó. Đồng thời, ta cũng cần giữ lại mọi object được tham chiếu bởi các field của instance. Hầu hết các object còn sống nhưng không phải root đều có thể truy cập được vì có một instance nào đó tham chiếu tới chúng qua field. May mắn là ta đã có sẵn hàm `markTable()` tiện lợi để việc tracing này trở nên dễ dàng.

Ít quan trọng hơn nhưng vẫn cần thiết là việc in ra.

^code print-instance (1 before, 1 after)

<span name="print">Một</span> instance sẽ in ra tên của nó kèm theo chữ “instance”. (Phần “instance” chủ yếu để class và instance không in ra giống hệt nhau.)

<aside name="print">

Hầu hết các ngôn ngữ hướng đối tượng cho phép class định nghĩa một dạng method `toString()` để chỉ định cách các instance của nó được chuyển thành chuỗi và in ra. Nếu Lox bớt “đồ chơi” hơn, tôi cũng muốn hỗ trợ điều đó.

</aside>

Phần thú vị thực sự diễn ra ở interpreter. Lox không có từ khóa `new` đặc biệt. Cách để tạo một instance của class là gọi chính class đó như thể nó là một hàm. Runtime đã hỗ trợ lời gọi hàm, và nó kiểm tra kiểu của object được gọi để đảm bảo người dùng không cố gọi một số hoặc kiểu không hợp lệ khác.

Chúng ta mở rộng phần kiểm tra runtime đó với một case mới.

^code call-class (1 before, 1 after)

Nếu giá trị được gọi — object thu được khi đánh giá biểu thức bên trái dấu ngoặc đơn mở — là một class, thì ta xử lý nó như một lời gọi constructor. Ta <span name="args">tạo</span> một instance mới của class được gọi và lưu kết quả lên stack.

<aside name="args">

Hiện tại, ta bỏ qua mọi argument được truyền vào lời gọi. Chúng ta sẽ quay lại đoạn code này trong [chương tiếp theo](methods-and-initializers.html) khi thêm hỗ trợ cho initializer.

</aside>

Chúng ta đã tiến thêm một bước. Giờ ta có thể định nghĩa class và tạo instance của chúng.

```lox
class Brioche {}
print Brioche();
```

Lưu ý dấu ngoặc đơn sau `Brioche` ở dòng thứ hai. Lệnh này sẽ in ra  
“Brioche instance”.

## Biểu thức Get & Set

Cách biểu diễn object cho instance của chúng ta đã có thể lưu trữ trạng thái, nên việc còn lại chỉ là cung cấp chức năng đó cho người dùng. Field được truy cập và thay đổi thông qua các biểu thức get và set. Giữ truyền thống, Lox dùng cú pháp “dấu chấm” kinh điển:

```lox
eclair.filling = "pastry creme";
print eclair.filling;
```

Dấu chấm — hay “full stop” cho các bạn nói tiếng Anh — hoạt động <span name="sort">gần giống</span> như một toán tử infix. Có một biểu thức ở bên trái được đánh giá trước và tạo ra một instance. Sau đó là dấu `.` và một tên field. Vì có một toán hạng đứng trước, ta gắn nó vào bảng parse như một biểu thức infix.

<aside name="sort">

Tôi nói “gần giống” vì phần bên phải sau dấu `.` không phải là một biểu thức, mà là một identifier duy nhất, có ngữ nghĩa được xử lý bởi chính biểu thức get hoặc set. Thực ra nó gần giống một biểu thức postfix hơn.

</aside>

^code table-dot (1 before, 1 after)

Giống như các ngôn ngữ khác, toán tử `.` có độ ưu tiên cao, ngang với dấu ngoặc đơn trong lời gọi hàm. Sau khi parser đọc token dấu chấm, nó sẽ chuyển sang một hàm parse mới.

^code compile-dot

Parser mong đợi tìm thấy một tên <span name="prop">property</span> ngay sau dấu chấm. Ta nạp lexeme của token đó vào constant table dưới dạng string để tên này có thể được truy cập ở runtime.

<aside name="prop">

Compiler dùng “property” thay vì “field” ở đây vì, nhớ rằng, Lox cũng cho phép bạn dùng cú pháp dấu chấm để truy cập một method mà không gọi nó. “Property” là thuật ngữ chung để chỉ bất kỳ thực thể có tên nào bạn có thể truy cập trên một instance. Field là tập con của property, được hỗ trợ bởi trạng thái của instance.

</aside>

Chúng ta có hai dạng biểu thức mới — getter và setter — và cả hai đều được xử lý trong cùng một hàm này. Nếu thấy dấu bằng ngay sau tên field, chắc chắn đó là một biểu thức set, gán giá trị cho field. Nhưng ta không phải lúc nào cũng *cho phép* compile dấu bằng sau field. Xem ví dụ:

```lox
a + b.c = 3
```

Theo grammar của Lox, đây là cú pháp không hợp lệ, nghĩa là bản cài đặt Lox của chúng ta bắt buộc phải phát hiện và báo lỗi. Nếu `dot()` âm thầm parse phần `= 3`, chúng ta sẽ diễn giải sai code như thể người dùng đã viết:

```lox
a + (b.c = 3)
```

Vấn đề là phía `=` của một biểu thức set có độ ưu tiên thấp hơn nhiều so với phần `.`. Parser có thể gọi `dot()` trong một ngữ cảnh có độ ưu tiên quá cao để cho phép setter xuất hiện. Để tránh việc cho phép sai, ta chỉ parse và compile phần dấu bằng khi `canAssign` là true. Nếu gặp token dấu bằng khi `canAssign` là false, `dot()` sẽ bỏ qua và trả về. Khi đó, compiler sẽ quay ngược lại lên `parsePrecedence()`, dừng lại ở dấu `=` bất ngờ vẫn đang là token kế tiếp và báo lỗi.

Nếu chúng ta tìm thấy dấu `=` trong một ngữ cảnh *được* phép, thì sẽ compile biểu thức theo sau. Sau đó, ta sinh ra một instruction mới <span name="set">`OP_SET_PROPERTY`</span>. Instruction này nhận một operand duy nhất là chỉ số của tên property trong constant table. Nếu không compile biểu thức set, ta mặc định đó là getter và sinh ra instruction `OP_GET_PROPERTY`, cũng nhận operand là tên property.

<aside name="set">

Bạn không thể *set* một property không phải field, nên instruction này lẽ ra có thể đặt tên là `OP_SET_FIELD`, nhưng tôi muốn giữ đồng nhất với instruction get cho đẹp.

</aside>

Giờ là lúc định nghĩa hai instruction mới này.

^code property-ops (1 before, 1 after)

Và thêm hỗ trợ giải mã (disassemble) chúng:

^code disassemble-property-ops (1 before, 1 after)

### Execute biểu thức getter & setter

Chuyển sang runtime, ta sẽ bắt đầu với biểu thức get vì chúng đơn giản hơn một chút.

^code interpret-get-property (1 before, 1 after)

Khi interpreter gặp instruction này, biểu thức bên trái dấu chấm đã được execute và instance kết quả đang nằm trên đỉnh stack. Ta đọc tên field từ constant pool và tra cứu trong bảng field của instance. Nếu hash table chứa một entry với tên đó, ta pop instance và push giá trị của entry đó làm kết quả.

Tất nhiên, field có thể không tồn tại. Trong Lox, chúng ta định nghĩa đây là một runtime error. Vì vậy, ta thêm một bước kiểm tra và dừng nếu gặp trường hợp này.

^code get-undefined (3 before, 2 after)

<span name="field">Có</span> một trường hợp lỗi khác cần xử lý mà có lẽ bạn đã nhận ra. Đoạn code trên giả định rằng biểu thức bên trái dấu chấm thực sự trả về một ObjInstance. Nhưng không có gì ngăn người dùng viết như sau:

```lox
var obj = "not an instance";
print obj.field;
```

Chương trình của người dùng là sai, nhưng VM vẫn phải xử lý một cách “êm đẹp”. Hiện tại, nó sẽ hiểu nhầm các bit của ObjString thành ObjInstance và… tôi cũng không biết, có thể “bốc cháy” hoặc làm gì đó chắc chắn không êm đẹp.

Trong Lox, chỉ instance mới được phép có field. Bạn không thể gắn field vào string hoặc number. Vì vậy, ta cần kiểm tra giá trị có phải là instance trước khi truy cập bất kỳ field nào của nó.

<aside name="field">

Lox *có thể* hỗ trợ thêm field vào các giá trị thuộc kiểu khác. Đây là ngôn ngữ của chúng ta và ta có thể làm điều mình muốn. Nhưng khả năng cao đây là một ý tưởng tồi. Nó làm phức tạp đáng kể phần cài đặt theo cách ảnh hưởng xấu đến hiệu năng — ví dụ, string interning sẽ khó hơn nhiều.

Ngoài ra, nó còn đặt ra những câu hỏi ngữ nghĩa rắc rối về equality và identity của giá trị. Nếu tôi gắn một field vào số `3`, thì kết quả của `1 + 2` có field đó không? Nếu có, implementation sẽ theo dõi điều đó thế nào? Nếu không, hai “số ba” kết quả đó có còn được coi là bằng nhau không?

</aside>

^code get-not-instance (1 before, 1 after)

Nếu giá trị trên stack không phải là instance, ta báo runtime error và thoát an toàn.

Tất nhiên, biểu thức get sẽ không hữu ích lắm nếu không instance nào có field. Để làm được điều đó, ta cần setter.

^code interpret-set-property (2 before, 1 after)

Phần này phức tạp hơn một chút so với `OP_GET_PROPERTY`. Khi instruction này chạy, đỉnh stack là instance có field đang được gán, và ngay trên nó là giá trị cần lưu. Giống như trước, ta đọc operand của instruction và tìm string tên field. Dùng tên đó, ta lưu giá trị trên đỉnh stack vào bảng field của instance.

Sau đó là một chút “ảo thuật” với <span name="stack">stack</span>. Ta pop giá trị vừa lưu, rồi pop instance, và cuối cùng push lại giá trị đó. Nói cách khác, ta loại bỏ *phần tử thứ hai* từ đỉnh stack nhưng giữ nguyên phần tử trên cùng. Setter bản thân nó là một biểu thức mà kết quả là giá trị được gán, nên ta cần giữ giá trị đó lại trên stack. Ý tôi là như thế này:


<aside name="stack">

Các thao tác trên stack diễn ra như sau:

<img src="image/classes-and-instances/stack.png" alt="Pop hai giá trị rồi push lại giá trị đầu tiên lên stack."/>

</aside>

```lox
class Toast {}
var toast = Toast();
print toast.jam = "grape"; // In ra "grape".
```

Khác với khi đọc một field, chúng ta không cần lo về việc hash table không chứa field đó. Một setter sẽ ngầm tạo field nếu cần. Tuy nhiên, ta vẫn cần xử lý trường hợp người dùng cố gắng lưu một field vào một giá trị không phải instance.

^code set-not-instance (1 before, 1 after)

Giống hệt như với biểu thức get, ta kiểm tra kiểu của giá trị và báo runtime error nếu nó không hợp lệ. Và như vậy, phần “có trạng thái” trong hỗ trợ lập trình hướng đối tượng của Lox đã hoàn thiện. Hãy thử nhé:

```lox
class Pair {}

var pair = Pair();
pair.first = 1;
pair.second = 2;
print pair.first + pair.second; // 3.
```

Điều này chưa thực sự mang cảm giác *hướng đối tượng*. Nó giống như một biến thể kỳ lạ, kiểu dynamic của C, nơi object chỉ là những “túi dữ liệu” lỏng lẻo giống struct. Kiểu như một ngôn ngữ thủ tục dynamic. Nhưng đây là một bước tiến lớn về khả năng biểu đạt. Bản cài đặt Lox của chúng ta giờ cho phép người dùng tự do gom dữ liệu thành các đơn vị lớn hơn. Trong chương tiếp theo, chúng ta sẽ “thổi hồn” vào những khối dữ liệu bất động đó.

<div class="challenges">

## Thử thách

1.  Việc cố truy cập một field không tồn tại trên object sẽ ngay lập tức dừng toàn bộ VM. Người dùng không có cách nào để phục hồi từ runtime error này, cũng như không có cách nào để kiểm tra xem một field có tồn tại *trước khi* cố truy cập nó. Người dùng phải tự đảm bảo rằng chỉ đọc các field hợp lệ.

    Các ngôn ngữ dynamic khác xử lý field bị thiếu như thế nào? Bạn nghĩ Lox nên làm gì? Hãy cài đặt giải pháp của bạn.

2.  Field được truy cập tại runtime bằng *tên chuỗi* của chúng. Nhưng tên đó luôn phải xuất hiện trực tiếp trong mã nguồn dưới dạng *identifier token*. Một chương trình không thể tạo ra một giá trị string một cách mệnh lệnh rồi dùng nó làm tên field. Bạn có nghĩ là nên cho phép không? Hãy nghĩ ra một tính năng ngôn ngữ cho phép điều đó và cài đặt nó.

3.  Ngược lại, Lox không có cách nào để *xóa* một field khỏi instance. Bạn có thể gán giá trị `nil` cho field, nhưng entry trong hash table vẫn còn đó. Các ngôn ngữ khác xử lý việc này ra sao? Hãy chọn và cài đặt một chiến lược cho Lox.

4.  Vì field được truy cập theo tên tại runtime, việc làm việc với trạng thái instance là chậm. Về mặt kỹ thuật, đây là một thao tác thời gian hằng số — nhờ hash table — nhưng hệ số hằng số lại khá lớn. Đây là một nguyên nhân chính khiến ngôn ngữ dynamic chậm hơn ngôn ngữ static.

    Các bản cài đặt tinh vi của ngôn ngữ dynamic xử lý và tối ưu điều này như thế nào?

</div>