> Bạn có thể chọn bạn bè nhưng chắc chắn không thể chọn gia đình mình, và họ vẫn là máu mủ của bạn dù bạn có thừa nhận hay không, và trông bạn sẽ thật ngớ ngẩn nếu giả vờ như không biết họ.
>
> <cite>Harper Lee, <em>Giết Con Chim Nhại</em></cite>

Đây là chương cuối cùng mà ta thêm chức năng mới cho VM. Ta đã nhét gần như toàn bộ ngôn ngữ Lox vào đây rồi. Việc còn lại chỉ là kế thừa method và gọi method của superclass. Sau chương này, ta còn [một chương nữa](optimization.html), nhưng nó không giới thiệu hành vi mới nào. Nó <span name="faster">chỉ</span> làm cho những thứ hiện có chạy nhanh hơn. Hoàn thành chương này, bạn sẽ có một bản hiện thực Lox hoàn chỉnh.

<aside name="faster">

Từ “chỉ” ở đây không có nghĩa là việc tăng tốc không quan trọng! Xét cho cùng, toàn bộ mục đích của chiếc máy ảo thứ hai này là để có hiệu năng tốt hơn jlox. Bạn thậm chí có thể nói rằng *tất cả* mười lăm chương vừa qua đều là “tối ưu hóa”.

</aside>

Một số nội dung trong chương này sẽ khiến bạn nhớ đến jlox. Cách ta xử lý lời gọi `super` gần như giống hệt, chỉ là nhìn qua cơ chế lưu trữ trạng thái trên stack phức tạp hơn của clox. Nhưng lần này, ta có một cách hoàn toàn khác, nhanh hơn nhiều, để xử lý lời gọi method được kế thừa.

## Kế thừa Method

Ta sẽ bắt đầu với kế thừa method vì nó đơn giản hơn. Nhắc lại một chút, cú pháp kế thừa trong Lox trông như sau:

```lox
class Doughnut {
  cook() {
    print "Dunk in the fryer.";
  }
}

class Cruller < Doughnut {
  finish() {
    print "Glaze with icing.";
  }
}
```

Ở đây, class `Cruller` kế thừa từ `Doughnut` và vì thế, các instance của `Cruller` cũng kế thừa method `cook()`. Tôi không biết tại sao mình lại phải giải thích dài dòng thế này. Bạn biết cách kế thừa hoạt động rồi. Bắt đầu compile cú pháp mới thôi.

^code compile-superclass (2 before, 1 after)

Sau khi compile tên class, nếu token tiếp theo là `<`, tức là ta đã gặp mệnh đề superclass. Ta đọc token identifier của superclass, rồi gọi `variable()`. Hàm này nhận token vừa đọc, coi nó như một biến, và sinh code để load giá trị của biến đó. Nói cách khác, nó tìm superclass theo tên và đẩy nó lên stack.

Sau đó, ta gọi `namedVariable()` để load subclass đang kế thừa lên stack, rồi theo sau là lệnh `OP_INHERIT`. Lệnh này kết nối superclass với subclass mới. Ở chương trước, ta đã định nghĩa lệnh `OP_METHOD` để thay đổi một class object hiện có bằng cách thêm method vào method table của nó. Lệnh `OP_INHERIT` cũng tương tự — nó nhận một class hiện có và áp dụng hiệu ứng kế thừa cho nó.

Trong ví dụ trước, khi compiler xử lý đoạn cú pháp:

```lox
class Cruller < Doughnut {
```

Kết quả là bytecode như sau:

<img src="image/superclasses/inherit-stack.png" alt="Chuỗi lệnh bytecode cho class Cruller kế thừa từ Doughnut." />

Trước khi hiện thực lệnh `OP_INHERIT` mới, ta cần phát hiện một trường hợp đặc biệt.

^code inherit-self (1 before, 1 after)

<span name="cycle">Một</span> class không thể là superclass của chính nó. Trừ khi bạn có trong tay một nhà vật lý hạt nhân “điên” và một chiếc DeLorean được độ cực mạnh, bạn không thể tự kế thừa chính mình.

<aside name="cycle">

Thú vị là, với cách ta hiện thực kế thừa method, tôi nghĩ cho phép vòng lặp kế thừa cũng không gây vấn đề gì trong clox. Nó sẽ chẳng làm được gì *có ích*, nhưng tôi không nghĩ nó sẽ gây crash hay vòng lặp vô hạn.

</aside>

### Execute kế thừa

Giờ đến lệnh mới.

^code inherit-op (1 before, 1 after)

Không có toán hạng nào cần lo. Hai giá trị ta cần — superclass và subclass — đều nằm trên stack. Điều đó khiến việc disassemble trở nên dễ dàng.

^code disassemble-inherit (1 before, 1 after)

Interpreter mới là nơi mọi thứ diễn ra.

^code interpret-inherit (1 before, 1 after)

Từ đỉnh stack trở xuống, ta có subclass rồi đến superclass. Ta lấy cả hai và thực hiện phần “inherit-y”. Đây là chỗ clox đi khác jlox. Trong interpreter đầu tiên, mỗi subclass lưu một tham chiếu tới superclass của nó. Khi truy cập method, nếu không tìm thấy trong method table của subclass, ta sẽ đệ quy qua chuỗi kế thừa, tìm trong method table của từng ancestor cho đến khi thấy.

Ví dụ, gọi `cook()` trên một instance của `Cruller` sẽ khiến jlox thực hiện hành trình này:

<img src="image/superclasses/jlox-resolve.png" alt="Việc resolve lời gọi cook() trong một instance của Cruller nghĩa là duyệt qua chuỗi superclass." />

Đó là rất nhiều việc phải làm tại thời điểm *gọi* method. Nó chậm, và tệ hơn, method kế thừa càng ở cao trong chuỗi ancestor thì càng chậm. Không phải là một câu chuyện hiệu năng hay ho.

Cách tiếp cận mới nhanh hơn nhiều. Khi subclass được khai báo, ta copy tất cả method của superclass xuống method table của subclass. Sau này, khi *gọi* method, bất kỳ method nào kế thừa từ superclass sẽ được tìm thấy ngay trong method table của subclass. Không cần thêm công việc runtime nào cho kế thừa cả. Ngay khi class được khai báo, mọi việc đã xong. Điều này có nghĩa là lời gọi method kế thừa nhanh y như lời gọi method bình thường — chỉ <span name="two">một</span> lần tra cứu hash table.

<img src="image/superclasses/clox-resolve.png" alt="Resolving a call to cook() in an instance of Cruller which has the method in its own method table." />

<aside name="two">

Thực ra là hai lần tra cứu hash table, tôi đoán vậy. Vì trước tiên ta phải đảm bảo rằng một field trên instance không che khuất method.

</aside>

Thỉnh thoảng tôi nghe kỹ thuật này được gọi là “copy-down inheritance” (kế thừa sao chép xuống). Nó đơn giản và nhanh, nhưng giống như hầu hết các tối ưu hóa, bạn chỉ có thể dùng nó trong một số điều kiện nhất định. Nó hoạt động trong Lox vì class trong Lox là *đóng*. Khi một khai báo class đã execute xong, tập hợp method của class đó sẽ không bao giờ thay đổi nữa.

Trong các ngôn ngữ như Ruby, Python và JavaScript, bạn có thể <span name="monkey">mở tung</span> một class hiện có và nhét thêm method mới vào hoặc thậm chí xóa chúng đi. Điều đó sẽ phá vỡ tối ưu hóa của ta, vì nếu những thay đổi đó xảy ra với một superclass *sau khi* khai báo subclass đã execute, subclass sẽ không nhận được các thay đổi đó. Điều này phá vỡ kỳ vọng của người dùng rằng kế thừa luôn phản ánh trạng thái hiện tại của superclass.

<aside name="monkey">

Như bạn có thể tưởng tượng, việc thay đổi tập hợp method mà một class định nghĩa một cách mệnh lệnh khi runtime có thể khiến việc suy luận về chương trình trở nên khó khăn. Đây là một công cụ rất mạnh, nhưng cũng rất nguy hiểm.

Những người thấy công cụ này có lẽ hơi *quá* nguy hiểm đã đặt cho nó cái tên không mấy hay ho là “monkey patching”, hoặc thậm chí là “duck punching” còn ít lịch sự hơn.

<img src="image/superclasses/monkey.png" alt="Một con khỉ đeo bịt mắt, tất nhiên rồi." />

</aside>

May mắn cho chúng ta (nhưng có lẽ không may cho những người thích tính năng này), Lox không cho phép bạn “vá khỉ” hay “đấm vịt”, nên ta có thể áp dụng tối ưu hóa này một cách an toàn.

Thế còn override method thì sao? Việc sao chép method của superclass vào method table của subclass có xung đột với method riêng của subclass không? May mắn là không. Ta phát sinh `OP_INHERIT` sau lệnh `OP_CLASS` tạo subclass nhưng trước khi bất kỳ khai báo method và lệnh `OP_METHOD` nào được compile. Tại thời điểm ta sao chép method của superclass xuống, method table của subclass vẫn trống. Bất kỳ method nào subclass override sẽ ghi đè lên các entry kế thừa trong bảng.

### Superclass không hợp lệ

Hiện thực của ta đơn giản và nhanh, đúng kiểu tôi thích cho code VM. Nhưng nó không mạnh mẽ. Không có gì ngăn người dùng kế thừa từ một object vốn chẳng phải class:

```lox
var NotClass = "So not a class";
class OhNo < NotClass {}
```

Rõ ràng, chẳng lập trình viên nào có lòng tự trọng lại viết như vậy, nhưng ta phải phòng ngừa những người dùng Lox tiềm năng không có lòng tự trọng. Một kiểm tra runtime đơn giản sẽ xử lý được.

^code inherit-non-class (1 before, 1 after)

Nếu giá trị ta load từ identifier trong mệnh đề superclass không phải là một ObjClass, ta báo lỗi runtime để cho người dùng biết ta nghĩ gì về họ và code của họ.

## Lưu trữ Superclass

Bạn có để ý rằng khi ta thêm kế thừa method, ta thực ra không hề thêm bất kỳ tham chiếu nào từ subclass tới superclass của nó không? Sau khi sao chép các method kế thừa xong, ta quên luôn superclass. Ta không cần giữ tham chiếu tới superclass, nên ta bỏ qua.

Điều đó sẽ không đủ để hỗ trợ lời gọi `super`. Vì subclass <span name="may">có thể</span> override method của superclass, ta cần có cách truy cập vào method table của superclass. Trước khi đi vào cơ chế đó, tôi muốn nhắc lại cho bạn cách lời gọi `super` được resolve tĩnh.

<aside name="may">

“Có thể” ở đây có lẽ chưa đủ mạnh. Nhiều khả năng method *đã* bị override. Nếu không thì tại sao bạn lại dùng `super` thay vì gọi trực tiếp?

</aside>

Quay lại những ngày tươi đẹp của jlox, tôi đã cho bạn xem [ví dụ hóc búa này](inheritance.html#semantics) để giải thích cách lời gọi `super` được phân giải:

```lox
class A {
  method() {
    print "A method";
  }
}

class B < A {
  method() {
    print "B method";
  }

  test() {
    super.method();
  }
}

class C < B {}

C().test();
```

Bên trong thân method `test()`, `this` là một instance của C. Nếu lời gọi `super` được resolve dựa trên superclass của *receiver*, thì ta sẽ tìm trong superclass của C, tức là B. Nhưng `super` được resolve dựa trên superclass của *class bao quanh nơi lời gọi super xảy ra*. Trong trường hợp này, ta đang ở method `test()` của B, nên superclass là A, và chương trình sẽ in ra `"A method"`.

Điều này có nghĩa là lời gọi `super` không được resolve một cách động dựa trên instance khi runtime. Superclass được dùng để tìm method là một thuộc tính tĩnh — gần như là lexical — của vị trí lời gọi xảy ra. Khi ta thêm kế thừa vào jlox, ta đã tận dụng khía cạnh tĩnh đó bằng cách lưu superclass trong cùng cấu trúc Environment mà ta dùng cho tất cả các scope lexical. Gần như thể interpreter nhìn chương trình trên như thế này:

```lox
class A {
  method() {
    print "A method";
  }
}

var Bs_super = A;
class B < A {
  method() {
    print "B method";
  }

  test() {
    runtimeSuperCall(Bs_super, "method");
  }
}

var Cs_super = B;
class C < B {}

C().test();
```

Mỗi subclass có một biến ẩn lưu tham chiếu tới superclass của nó. Bất cứ khi nào cần thực hiện một lời gọi `super`, ta sẽ truy cập superclass từ biến đó và yêu cầu runtime bắt đầu tìm method từ đó trở đi.

Ta sẽ áp dụng cách tiếp cận tương tự với clox. Khác biệt là thay vì dùng class Environment được cấp phát trên heap như jlox, ta có value stack và hệ thống upvalue của bytecode VM. Cơ chế hơi khác một chút, nhưng hiệu ứng tổng thể thì giống nhau.

### Biến local cho superclass

Compiler của ta vốn đã sinh code để load superclass lên stack. Thay vì để slot đó như một biến tạm, ta tạo một scope mới và biến nó thành một biến local.

^code superclass-variable (2 before, 2 after)

Việc tạo một lexical scope mới đảm bảo rằng nếu ta khai báo hai class trong cùng một scope, mỗi class sẽ có một slot local khác nhau để lưu superclass của nó. Vì ta luôn đặt tên biến này là `"super"`, nếu không tạo scope riêng cho mỗi subclass, các biến này sẽ bị trùng nhau.

Ta đặt tên biến là `"super"` vì cùng lý do ta dùng `"this"` làm tên biến local ẩn mà các biểu thức `this` sẽ resolve tới: `"super"` là một từ khóa, đảm bảo biến ẩn của compiler sẽ không bị trùng với biến do người dùng định nghĩa.

Điểm khác là khi compile biểu thức `this`, ta tiện lợi có sẵn một token với lexeme là `"this"`. Ở đây thì không may mắn như vậy. Thay vào đó, ta thêm một hàm helper nhỏ để tạo một token giả (synthetic token) cho một chuỗi <span name="constant">hằng</span> cho trước.

^code synthetic-token

<aside name="constant" class="bottom">

Tôi nói “chuỗi hằng” vì token không quản lý bộ nhớ cho lexeme của nó. Nếu ta cố dùng một chuỗi được cấp phát trên heap cho việc này, ta sẽ bị rò rỉ bộ nhớ vì nó sẽ không bao giờ được giải phóng. Nhưng bộ nhớ cho các chuỗi literal trong C thì nằm trong vùng dữ liệu hằng của file execute và không cần giải phóng, nên ta ổn.

</aside>

Vì ta đã mở một local scope cho biến superclass, ta cần đóng nó lại.

^code end-superclass-scope (1 before, 2 after)

Ta pop scope và loại bỏ biến `"super"` sau khi compile xong thân class và các method của nó. Cách này giúp biến đó khả dụng trong tất cả các method của subclass. Đây là một tối ưu hóa hơi vô nghĩa, nhưng ta chỉ tạo scope nếu *có* mệnh đề superclass. Do đó, ta chỉ cần đóng scope nếu có mệnh đề này.

Để theo dõi điều đó, ta có thể khai báo một biến local nhỏ trong `classDeclaration()`. Nhưng sớm thôi, các hàm khác trong compiler cũng sẽ cần biết class bao quanh có phải là subclass hay không. Vậy nên tốt hơn là giúp chính mình trong tương lai bằng cách lưu thông tin này vào một field trong ClassCompiler ngay bây giờ.

^code has-superclass (2 before, 1 after)

Khi khởi tạo một ClassCompiler, ta giả định nó không phải là subclass.

^code init-has-superclass (1 before, 1 after)

Sau đó, nếu ta thấy một mệnh đề superclass, ta biết mình đang compile một subclass.

^code set-has-superclass (1 before, 1 after)

Cơ chế này cho phép ta, tại runtime, truy cập object superclass của subclass bao quanh từ bên trong bất kỳ method nào của subclass — chỉ cần sinh code để load biến tên `"super"`. Biến đó là một local nằm ngoài thân method, nhưng hệ thống upvalue hiện có của VM cho phép capture biến local đó bên trong thân method hoặc thậm chí trong các hàm lồng bên trong method đó.

## Lời gọi `super`

Với phần hỗ trợ runtime đã sẵn sàng, ta có thể bắt tay vào hiện thực lời gọi `super`. Như thường lệ, ta sẽ đi từ front-end tới back-end, bắt đầu với cú pháp mới. Một lời gọi `super` <span name="last">bắt đầu</span> — tất nhiên rồi — bằng từ khóa `super`.

<aside name="last">

Đây rồi, bạn của tôi. Mục cuối cùng bạn sẽ thêm vào bảng parse.

</aside>

^code table-super (1 before, 1 after)

Khi expression parser gặp một token `super`, điều khiển sẽ nhảy tới một hàm parse mới, bắt đầu như sau:

^code super

Điều này khá khác so với cách ta compile biểu thức `this`. Không giống `this`, một token `super` <span name="token">không</span> phải là một expression độc lập. Thay vào đó, dấu chấm và tên method theo sau nó là những phần không thể tách rời của cú pháp. Tuy nhiên, danh sách đối số trong ngoặc đơn thì tách biệt. Giống như truy cập method thông thường, Lox cho phép lấy một tham chiếu tới method của superclass dưới dạng closure mà không cần gọi nó:

<aside name="token">

Câu hỏi giả định: Nếu một token `super` trần *là* một expression, nó sẽ evaluate ra loại object nào?

</aside>

```lox
class A {
  method() {
    print "A";
  }
}

class B < A {
  method() {
    var closure = super.method;
    closure(); // Prints "A".
  }
}
```

<aside name="two">

Thực ra là hai lần tra cứu hash table, tôi đoán vậy. Vì trước tiên ta phải đảm bảo rằng một field trên instance không che khuất method.

</aside>

Thỉnh thoảng tôi nghe kỹ thuật này được gọi là “copy-down inheritance” (kế thừa sao chép xuống). Nó đơn giản và nhanh, nhưng giống như hầu hết các tối ưu hóa, bạn chỉ có thể dùng nó trong một số điều kiện nhất định. Nó hoạt động trong Lox vì class trong Lox là *đóng*. Khi một khai báo class đã execute xong, tập hợp method của class đó sẽ không bao giờ thay đổi nữa.

Trong các ngôn ngữ như Ruby, Python và JavaScript, bạn có thể <span name="monkey">mở tung</span> một class hiện có và nhét thêm method mới vào hoặc thậm chí xóa chúng đi. Điều đó sẽ phá vỡ tối ưu hóa của ta, vì nếu những thay đổi đó xảy ra với một superclass *sau khi* khai báo subclass đã execute, subclass sẽ không nhận được các thay đổi đó. Điều này phá vỡ kỳ vọng của người dùng rằng kế thừa luôn phản ánh trạng thái hiện tại của superclass.

<aside name="monkey">

Như bạn có thể tưởng tượng, việc thay đổi tập hợp method mà một class định nghĩa một cách mệnh lệnh khi runtime có thể khiến việc suy luận về chương trình trở nên khó khăn. Đây là một công cụ rất mạnh, nhưng cũng rất nguy hiểm.

Những người thấy công cụ này có lẽ hơi *quá* nguy hiểm đã đặt cho nó cái tên không mấy hay ho là “monkey patching”, hoặc thậm chí là “duck punching” còn ít lịch sự hơn.

<img src="image/superclasses/monkey.png" alt="Một con khỉ đeo bịt mắt, tất nhiên rồi." />

</aside>

May mắn cho chúng ta (nhưng có lẽ không may cho những người thích tính năng này), Lox không cho phép bạn “vá khỉ” hay “đấm vịt”, nên ta có thể áp dụng tối ưu hóa này một cách an toàn.

Thế còn override method thì sao? Việc sao chép method của superclass vào method table của subclass có xung đột với method riêng của subclass không? May mắn là không. Ta phát sinh `OP_INHERIT` sau lệnh `OP_CLASS` tạo subclass nhưng trước khi bất kỳ khai báo method và lệnh `OP_METHOD` nào được compile. Tại thời điểm ta sao chép method của superclass xuống, method table của subclass vẫn trống. Bất kỳ method nào subclass override sẽ ghi đè lên các entry kế thừa trong bảng.

### Superclass không hợp lệ

Hiện thực của ta đơn giản và nhanh, đúng kiểu tôi thích cho code VM. Nhưng nó không mạnh mẽ. Không có gì ngăn người dùng kế thừa từ một object vốn chẳng phải class:

```lox
var NotClass = "So not a class";
class OhNo < NotClass {}
```

Rõ ràng, chẳng lập trình viên nào có lòng tự trọng lại viết như vậy, nhưng ta phải phòng ngừa những người dùng Lox tiềm năng không có lòng tự trọng. Một kiểm tra runtime đơn giản sẽ xử lý được.

^code inherit-non-class (1 before, 1 after)

Nếu giá trị ta load từ identifier trong mệnh đề superclass không phải là một ObjClass, ta báo lỗi runtime để cho người dùng biết ta nghĩ gì về họ và code của họ.

## Lưu trữ Superclass

Bạn có để ý rằng khi ta thêm kế thừa method, ta thực ra không hề thêm bất kỳ tham chiếu nào từ subclass tới superclass của nó không? Sau khi sao chép các method kế thừa xong, ta quên luôn superclass. Ta không cần giữ tham chiếu tới superclass, nên ta bỏ qua.

Điều đó sẽ không đủ để hỗ trợ lời gọi `super`. Vì subclass <span name="may">có thể</span> override method của superclass, ta cần có cách truy cập vào method table của superclass. Trước khi đi vào cơ chế đó, tôi muốn nhắc lại cho bạn cách lời gọi `super` được resolve tĩnh.

<aside name="may">

“Có thể” ở đây có lẽ chưa đủ mạnh. Nhiều khả năng method *đã* bị override. Nếu không thì tại sao bạn lại dùng `super` thay vì gọi trực tiếp?

</aside>

Quay lại những ngày tươi đẹp của jlox, tôi đã cho bạn xem [ví dụ hóc búa này](inheritance.html#semantics) để giải thích cách lời gọi `super` được phân giải:

```lox
class A {
  method() {
    print "A method";
  }
}

class B < A {
  method() {
    print "B method";
  }

  test() {
    super.method();
  }
}

class C < B {}

C().test();
```

Bên trong thân method `test()`, `this` là một instance của C. Nếu lời gọi `super` được resolve dựa trên superclass của *receiver*, thì ta sẽ tìm trong superclass của C, tức là B. Nhưng `super` được resolve dựa trên superclass của *class bao quanh nơi lời gọi super xảy ra*. Trong trường hợp này, ta đang ở method `test()` của B, nên superclass là A, và chương trình sẽ in ra `"A method"`.

Điều này có nghĩa là lời gọi `super` không được resolve một cách động dựa trên instance khi runtime. Superclass được dùng để tìm method là một thuộc tính tĩnh — gần như là lexical — của vị trí lời gọi xảy ra. Khi ta thêm kế thừa vào jlox, ta đã tận dụng khía cạnh tĩnh đó bằng cách lưu superclass trong cùng cấu trúc Environment mà ta dùng cho tất cả các scope lexical. Gần như thể interpreter nhìn chương trình trên như thế này:

Nói cách khác, Lox thực ra không có biểu thức `super` dạng *gọi hàm* (call expression), mà là biểu thức `super` dạng *truy cập* (access expression), và bạn có thể chọn gọi ngay nó nếu muốn. Vì vậy, khi compiler gặp một token `super`, ta đọc tiếp token `.` theo sau và sau đó tìm tên method. Method được tra cứu một cách động, nên ta dùng `identifierConstant()` để lấy lexeme của token tên method và lưu nó vào constant table, giống như ta làm với các biểu thức truy cập property.

Đây là những gì compiler làm sau khi đọc các token đó:

^code super-get (1 before, 1 after)

Để truy cập một *method của superclass* trên *instance hiện tại*, runtime cần cả receiver *và* superclass của class chứa method đó. Lệnh `namedVariable()` đầu tiên sinh code để tìm receiver hiện tại được lưu trong biến ẩn `"this"` và đẩy nó lên stack. Lệnh `namedVariable()` thứ hai sinh code để tìm superclass từ biến `"super"` của nó và đẩy lên trên cùng.

Cuối cùng, ta sinh một lệnh mới `OP_GET_SUPER` với toán hạng là chỉ số trong constant table của tên method. Nghe thì nhiều thứ phải nhớ, nên để dễ hình dung, hãy xem ví dụ chương trình này:

```lox
class Doughnut {
  cook() {
    print "Dunk in the fryer.";
    this.finish("sprinkles");
  }

  finish(ingredient) {
    print "Finish with " + ingredient;
  }
}

class Cruller < Doughnut {
  finish(ingredient) {
    // Không rắc sprinkles, luôn dùng icing.
    super.finish("icing");
  }
}
```

Bytecode được generated cho biểu thức `super.finish("icing")` trông và hoạt động như sau:

<img src="image/superclasses/super-instructions.png" alt="Chuỗi lệnh bytecode cho lời gọi super.finish()." />

Ba lệnh đầu tiên cung cấp cho runtime ba mảnh thông tin cần thiết để thực hiện truy cập `super`:

1.  Lệnh đầu tiên load **instance** lên stack.

2.  Lệnh thứ hai load **superclass nơi method được resolve**.

3.  Sau đó, lệnh `OP_GET_SUPER` mới mã hóa **tên method cần truy cập** dưới dạng toán hạng.

Các lệnh còn lại là bytecode thông thường để đánh giá danh sách đối số và gọi hàm.

Ta gần như đã sẵn sàng hiện thực lệnh `OP_GET_SUPER` mới trong interpreter. Nhưng trước khi làm vậy, compiler cần báo một số lỗi mà nó chịu trách nhiệm.

^code super-errors (1 before, 1 after)

Một lời gọi `super` chỉ có ý nghĩa bên trong thân của một method (hoặc trong một hàm lồng bên trong method), và chỉ trong method của một class có superclass. Ta phát hiện cả hai trường hợp này bằng cách dùng giá trị của `currentClass`. Nếu nó là `NULL` hoặc trỏ tới một class không có superclass, ta báo lỗi.

### Execute truy cập `super`

Giả sử người dùng không đặt biểu thức `super` ở nơi không được phép, code của họ sẽ đi từ compiler sang runtime. Ta có một lệnh mới.

^code get-super-op (1 before, 1 after)

Ta disassemble nó giống như các opcode khác có toán hạng là chỉ số constant table.

^code disassemble-get-super (1 before, 1 after)

Bạn có thể nghĩ sẽ phức tạp hơn, nhưng việc execute lệnh mới này khá giống với việc execute truy cập property thông thường.

^code interpret-get-super (1 before, 1 after)

Giống như với property, ta đọc tên method từ constant table. Sau đó, ta truyền nó vào `bindMethod()`, hàm này sẽ tìm method trong method table của class được chỉ định và tạo một ObjBoundMethod để gắn closure kết quả với instance hiện tại.

Điểm <span name="field">khác biệt</span> chính là *class nào* ta truyền vào `bindMethod()`. Với truy cập property thông thường, ta dùng chính class của ObjInstance, điều này cho ta dynamic dispatch như mong muốn. Với lời gọi `super`, ta không dùng class của instance. Thay vào đó, ta dùng superclass đã được resolve tĩnh của class chứa method, mà compiler đã đảm bảo tiện lợi đặt sẵn trên đỉnh stack chờ ta.

Ta pop superclass đó và truyền nó vào `bindMethod()`, hàm này sẽ bỏ qua đúng cách bất kỳ method override nào trong các subclass nằm giữa superclass đó và class của instance. Nó cũng bao gồm đúng cách mọi method mà superclass kế thừa từ bất kỳ superclass nào của *nó*.

Phần còn lại của hành vi thì giống nhau. Pop superclass sẽ để lại instance ở đỉnh stack. Khi `bindMethod()` thành công, nó pop instance và push bound method mới. Nếu không, nó báo lỗi runtime và trả về `false`. Trong trường hợp đó, ta dừng interpreter.

<aside name="field">

Một điểm khác so với `OP_GET_PROPERTY` là ta không cố tìm field che khuất trước. Field không được kế thừa, nên biểu thức `super` luôn resolve tới method.

Nếu Lox là một ngôn ngữ dựa trên prototype và dùng *delegation* thay vì *inheritance*, thì thay vì một *class* kế thừa từ một *class* khác, các instance sẽ kế thừa từ (hay “ủy quyền cho”) các instance khác. Khi đó, field *có thể* được kế thừa, và ta sẽ cần kiểm tra chúng ở đây.

</aside>

### Lời gọi `super` nhanh hơn

Giờ ta đã có thể truy cập method của superclass. Và vì object trả về là một ObjBoundMethod mà bạn có thể gọi, nên ta cũng đã có lời gọi `super` hoạt động. Giống như chương trước, ta đã đạt tới điểm mà VM của ta có ngữ nghĩa đầy đủ và chính xác.

Nhưng, cũng như chương trước, nó khá chậm. Một lần nữa, ta lại cấp phát trên heap một ObjBoundMethod cho mỗi lời gọi `super` dù rằng hầu hết thời gian, lệnh tiếp theo ngay sau đó là `OP_CALL` sẽ lập tức “mở” bound method đó, gọi nó, rồi bỏ đi. Thực tế, điều này còn có khả năng xảy ra cao hơn với lời gọi `super` so với lời gọi method thông thường. Ít nhất với lời gọi method, vẫn có khả năng người dùng thực sự đang gọi một hàm được lưu trong một field. Với lời gọi `super`, bạn *luôn* tìm method. Câu hỏi duy nhất là bạn có gọi nó ngay lập tức hay không.

Compiler hoàn toàn có thể tự trả lời câu hỏi đó nếu nó thấy một dấu ngoặc đơn ngay sau tên method của superclass, nên ta sẽ thực hiện cùng một tối ưu hóa như với lời gọi method. Bỏ hai dòng code load superclass và phát sinh `OP_GET_SUPER`, thay bằng đoạn này:

^code super-invoke (1 before, 1 after)

Giờ trước khi phát sinh bất kỳ thứ gì, ta tìm danh sách đối số trong ngoặc đơn. Nếu tìm thấy, ta compile nó. Sau đó ta load superclass. Tiếp theo, ta phát sinh một lệnh mới `OP_SUPER_INVOKE`. Lệnh <span name="superinstruction">superinstruction</span> này kết hợp hành vi của `OP_GET_SUPER` và `OP_CALL`, nên nó nhận hai toán hạng: chỉ số trong constant table của tên method cần tìm và số lượng đối số cần truyền vào.

<aside name="superinstruction">

Đây là một *super* superinstruction đúng nghĩa, nếu bạn hiểu ý tôi. Tôi… xin lỗi vì trò đùa tệ này.

</aside>

Ngược lại, nếu không tìm thấy `(`, ta tiếp tục compile biểu thức như một truy cập `super` như trước và phát sinh `OP_GET_SUPER`.

Trôi xuống đường ống compile, điểm dừng đầu tiên của ta là một lệnh mới.

^code super-invoke-op (1 before, 1 after)

Và ngay sau đó là phần hỗ trợ disassembler của nó.

^code disassemble-super-invoke (1 before, 1 after)

Một lệnh gọi `super` có cùng tập toán hạng như `OP_INVOKE`, nên ta tái sử dụng cùng helper để disassemble nó. Cuối cùng, đường ống đưa ta vào interpreter.

^code interpret-super-invoke (2 before, 1 after)

Đoạn code này về cơ bản là hiện thực của `OP_INVOKE` trộn thêm một chút `OP_GET_SUPER`. Tuy nhiên, có vài khác biệt về cách tổ chức stack. Với một lời gọi `super` chưa tối ưu, superclass sẽ bị pop và thay thế bằng ObjBoundMethod của hàm đã resolve *trước khi* các đối số của lời gọi được execute. Điều này đảm bảo rằng khi `OP_CALL` được execute, bound method nằm *dưới* danh sách đối số, đúng vị trí runtime mong đợi cho một lời gọi closure.

Với lệnh tối ưu hóa, mọi thứ được xáo trộn một chút:

<img src="image/superclasses/super-invoke.png" class="wide" alt="Chuỗi lệnh bytecode cho lời gọi super.finish() dùng OP_SUPER_INVOKE." />

Giờ việc resolve method của superclass là một phần của *invocation*, nên các đối số cần phải có sẵn trên stack tại thời điểm ta tìm method. Điều này có nghĩa là object superclass nằm trên cùng của các đối số.

Ngoài ra, hành vi gần như giống hệt `OP_GET_SUPER` theo sau bởi `OP_CALL`. Đầu tiên, ta lấy tên method và số lượng đối số từ toán hạng. Sau đó, ta pop superclass khỏi đỉnh stack để tìm method trong method table của nó. Việc này tiện lợi để lại stack ở trạng thái sẵn sàng cho một lời gọi method.

Ta truyền superclass, tên method và số lượng đối số vào hàm `invokeFromClass()` hiện có. Hàm này tìm method được chỉ định trên class được chỉ định và cố gắng tạo một lời gọi tới nó với arity đã cho. Nếu không tìm thấy method, nó trả về `false` và ta thoát interpreter. Ngược lại, `invokeFromClass()` sẽ push một CallFrame mới lên call stack cho closure của method. Điều đó làm mất hiệu lực con trỏ CallFrame đã cache của interpreter, nên ta làm mới `frame`.

## Một máy ảo hoàn chỉnh

Hãy nhìn lại những gì ta đã tạo ra. Theo tôi đếm, ta đã viết khoảng 2.500 dòng C khá sạch và rõ ràng. Chương trình nhỏ bé đó chứa một hiện thực hoàn chỉnh của ngôn ngữ Lox — khá là cấp cao! — với một bảng độ ưu tiên đầy đủ các loại biểu thức và một bộ câu lệnh điều khiển luồng. Ta đã hiện thực biến, hàm, closure, class, field, method và kế thừa.

Ấn tượng hơn nữa, hiện thực của ta có thể chạy trên bất kỳ nền tảng nào có compiler C, và đủ nhanh cho việc sử dụng thực tế. Ta có một compiler bytecode một-pass, một interpreter máy ảo chặt chẽ cho tập lệnh nội bộ, các biểu diễn object gọn nhẹ, một stack để lưu biến mà không cần cấp phát heap, và một garbage collector chính xác.

Nếu bạn đi tìm hiểu các hiện thực của Lua, Python hoặc Ruby, bạn sẽ ngạc nhiên bởi có bao nhiêu thứ giờ đây trông quen thuộc. Bạn đã thực sự nâng cấp kiến thức của mình về cách ngôn ngữ lập trình hoạt động, từ đó hiểu sâu hơn về lập trình. Giống như trước đây bạn là tay đua xe, và giờ bạn có thể mở nắp capo và tự sửa động cơ.

Bạn có thể dừng ở đây nếu muốn. Hai hiện thực Lox mà bạn có giờ đã hoàn chỉnh và đầy đủ tính năng. Bạn đã chế tạo chiếc xe và có thể lái nó đi bất cứ đâu. Nhưng nếu bạn muốn vui hơn nữa với việc tinh chỉnh để đạt hiệu năng cao hơn trên đường đua, vẫn còn một chương nữa. Chúng ta sẽ không thêm khả năng mới nào, nhưng sẽ áp dụng một vài tối ưu hóa kinh điển để vắt thêm hiệu năng. Nếu nghe thú vị, [hãy đọc tiếp](optimization.html)...

## Thử thách

1.  Một nguyên tắc của lập trình hướng đối tượng là một class phải đảm bảo các object mới được tạo ra ở trạng thái hợp lệ. Trong Lox, điều đó có nghĩa là định nghĩa một initializer để gán giá trị cho các field của instance. Kế thừa làm phức tạp các bất biến này vì instance phải ở trạng thái hợp lệ theo tất cả các class trong chuỗi kế thừa của object.

    Phần dễ là nhớ gọi `super.init()` trong mỗi phương thức `init()` của subclass. Phần khó hơn là các field. Không có gì ngăn hai class trong chuỗi kế thừa vô tình dùng cùng một tên field. Khi điều này xảy ra, chúng sẽ ghi đè field của nhau và có thể khiến instance rơi vào trạng thái hỏng.

    Nếu Lox là ngôn ngữ của bạn, bạn sẽ xử lý điều này thế nào, hoặc có xử lý không? Nếu bạn muốn thay đổi ngôn ngữ, hãy hiện thực thay đổi đó.

2.  Tối ưu hóa “copy-down inheritance” của ta chỉ hợp lệ vì Lox không cho phép bạn sửa đổi method của một class sau khi nó được khai báo. Điều này có nghĩa là ta không phải lo việc các method đã sao chép trong subclass bị lệch so với các thay đổi sau này ở superclass.

    Các ngôn ngữ khác, như Ruby, *có* cho phép class được sửa đổi sau đó. Các hiện thực của những ngôn ngữ như vậy hỗ trợ việc sửa đổi class mà vẫn giữ cho việc tìm method hiệu quả bằng cách nào?

3.  Trong [chương về kế thừa của jlox](inheritance.html), ta đã có một thử thách để hiện thực cách tiếp cận của ngôn ngữ BETA đối với việc override method. Hãy giải lại thử thách đó, nhưng lần này trong clox. Đây là mô tả của thử thách trước:

    Trong Lox, cũng như hầu hết các ngôn ngữ hướng đối tượng khác, khi tìm một method, ta bắt đầu từ đáy của cây kế thừa và đi ngược lên — method của subclass được ưu tiên hơn method của superclass. Để gọi method của superclass từ bên trong một method override, bạn dùng `super`.

    Ngôn ngữ [BETA](https://beta.cs.au.dk/) lại [tiếp cận ngược lại](http://journal.stuffwithstuff.com/2012/12/19/the-impoliteness-of-overriding-methods/). Khi bạn gọi một method, nó bắt đầu từ *đỉnh* của cây kế thừa và đi *xuống*. Method của superclass sẽ thắng method của subclass. Để gọi method của subclass, method của superclass có thể gọi `inner`, thứ giống như nghịch đảo của `super`. Nó sẽ nối tiếp xuống method tiếp theo trong cây kế thừa.

    Method của superclass kiểm soát khi nào và ở đâu subclass được phép tinh chỉnh hành vi của nó. Nếu method của superclass không gọi `inner` chút nào, thì subclass không có cách nào để override hoặc thay đổi hành vi của superclass.

    Hãy bỏ hành vi override và `super` hiện tại của Lox, và thay thế bằng ngữ nghĩa của BETA. Tóm lại:

    *   Khi gọi một method trên một class, method *cao nhất* trong chuỗi kế thừa của class đó sẽ được ưu tiên.

    *   Bên trong thân của một method, một lời gọi `inner` sẽ tìm method cùng tên trong subclass gần nhất dọc theo chuỗi kế thừa, nằm giữa class chứa `inner` và class của `this`. Nếu không có method khớp, lời gọi `inner` sẽ không làm gì cả.

    Ví dụ:

    ```lox
    class Doughnut {
      cook() {
        print "Fry until golden brown.";
        inner();
        print "Place in a nice box.";
      }
    }

    class BostonCream < Doughnut {
      cook() {
        print "Pipe full of custard and coat with chocolate.";
      }
    }

    BostonCream().cook();
    ```

    Kết quả in ra sẽ là:

    ```text
    Fry until golden brown.
    Pipe full of custard and coat with chocolate.
    Place in a nice box.
    ```

    Vì clox không chỉ là hiện thực Lox, mà còn hướng tới hiệu năng tốt, nên lần này hãy thử giải bài toán với mục tiêu tối ưu hiệu suất.