
> Khi bạn đã ở trên sàn nhảy, thì chẳng còn gì khác để làm ngoài việc nhảy.
>
> <cite>Umberto Eco, <em>The Mysterious Flame of Queen Loana</em></cite>

Đã đến lúc virtual machine của chúng ta thổi hồn vào những object mới chớm hình thành bằng hành vi. Điều đó có nghĩa là method và lời gọi method. Và vì initializer cũng là một dạng method đặc biệt, nên chúng ta sẽ làm cả chúng nữa.

Tất cả những điều này đều là “lãnh thổ quen thuộc” từ interpreter jlox trước đây. Điểm mới trong “chuyến đi” thứ hai này là một tối ưu quan trọng mà ta sẽ implement để khiến lời gọi method nhanh hơn gấp bảy lần so với hiệu năng cơ bản. Nhưng trước khi đến phần thú vị đó, ta cần làm cho những thứ cơ bản hoạt động trước đã.

## Khai báo Method

Ta không thể tối ưu lời gọi method trước khi có lời gọi method, và ta cũng không thể gọi method nếu chưa có method để gọi, nên ta sẽ bắt đầu với phần khai báo.

### Biểu diễn method

Thông thường ta sẽ bắt đầu ở compiler, nhưng lần này hãy xử lý phần object model trước. Biểu diễn runtime cho method trong clox tương tự như trong jlox. Mỗi class lưu một hash table các method. Key là tên method, và mỗi value là một ObjClosure cho phần thân method.

^code class-methods (3 before, 1 after)

Một class mới tinh bắt đầu với một bảng method rỗng.

^code init-methods (1 before, 1 after)

Struct ObjClass sở hữu vùng nhớ cho bảng này, nên khi memory manager giải phóng một class, bảng này cũng cần được free.

^code free-methods (1 before, 1 after)

Nói đến memory manager, GC cần trace qua class để vào bảng method. Nếu một class vẫn còn reachable (thường là qua một instance nào đó), thì tất cả method của nó chắc chắn cũng cần được giữ lại.

^code mark-methods (1 before, 1 after)

Ta dùng hàm `markTable()` sẵn có, hàm này sẽ trace qua key string và value trong mỗi entry của bảng.

Việc lưu method của class khá quen thuộc nếu bạn đến từ jlox. Phần khác biệt là cách bảng này được “lấp đầy”. Interpreter trước đây có thể truy cập toàn bộ AST node cho khai báo class và tất cả method bên trong nó. Ở runtime, interpreter chỉ việc duyệt qua danh sách khai báo đó.

Giờ đây, mọi thông tin mà compiler muốn chuyển sang runtime đều phải “len” qua giao diện của một chuỗi bytecode instruction phẳng. Làm sao để ta lấy một khai báo class — vốn có thể chứa một tập method lớn tùy ý — và biểu diễn nó thành bytecode? Hãy nhảy sang compiler để tìm hiểu.

### Compile khai báo method

Chương trước để lại cho ta một compiler có thể parse class nhưng chỉ cho phép phần thân rỗng. Giờ ta sẽ chèn thêm một chút code để compile một loạt khai báo method giữa hai dấu ngoặc nhọn.

^code class-body (1 before, 1 after)

Lox không có khai báo field, nên bất cứ thứ gì trước dấu ngoặc nhọn đóng ở cuối thân class đều phải là method. Ta dừng compile method khi gặp dấu ngoặc nhọn cuối cùng hoặc khi đến cuối file. Kiểm tra thứ hai đảm bảo compiler không bị kẹt trong vòng lặp vô hạn nếu người dùng quên dấu ngoặc nhọn đóng.

Điểm khó khi compile khai báo class là một class có thể khai báo bất kỳ số lượng method nào. Bằng cách nào đó, runtime cần tra cứu và bind tất cả chúng. Sẽ là quá nhiều nếu nhét hết vào một instruction `OP_CLASS`. Thay vào đó, bytecode mà ta tạo ra cho một khai báo class sẽ chia quá trình này thành một <span name="series">*chuỗi*</span> instruction. Compiler vốn đã phát sinh một instruction `OP_CLASS` để tạo ra một ObjClass rỗng. Sau đó nó phát sinh các instruction để lưu class vào một biến với tên của nó.

<aside name="series">

Ta đã làm điều gì đó tương tự với closure. Instruction `OP_CLOSURE` cần biết kiểu và chỉ số cho mỗi upvalue được capture. Ta mã hóa điều đó bằng một chuỗi pseudo-instruction theo sau instruction `OP_CLOSURE` chính — về cơ bản là một số lượng toán hạng biến đổi. VM sẽ xử lý tất cả byte bổ sung đó ngay khi interpret instruction `OP_CLOSURE`.

Ở đây, cách tiếp cận hơi khác vì từ góc nhìn của VM, mỗi instruction định nghĩa một method là một thao tác độc lập. Cả hai cách đều hoạt động. Một pseudo-instruction kích thước biến đổi có thể nhanh hơn đôi chút, nhưng khai báo class hiếm khi nằm trong hot loop, nên điều đó không quan trọng lắm.

</aside>

Giờ, với mỗi khai báo method, ta phát sinh một instruction `OP_METHOD` mới để thêm một method vào class đó. Khi tất cả instruction `OP_METHOD` đã execute, ta sẽ có một class hoàn chỉnh. Trong khi người dùng nhìn thấy khai báo class như một thao tác nguyên tử duy nhất, VM lại implement nó như một chuỗi thay đổi.

Để định nghĩa một method mới, VM cần ba thứ:

1.  Tên method.

2.  Closure cho phần thân method.

3.  Class để bind method vào.

Ta sẽ viết code compiler từng bước để xem cách tất cả những thứ này được chuyển đến runtime, bắt đầu từ đây:

^code method

Giống như `OP_GET_PROPERTY` và các instruction khác cần tên ở runtime, compiler thêm lexeme của token tên method vào constant table và nhận lại chỉ số trong bảng. Sau đó ta phát sinh một instruction `OP_METHOD` với chỉ số đó làm toán hạng. Đó là phần tên. Tiếp theo là phần thân method:

^code method-body (1 before, 1 after)

Ta dùng cùng helper `function()` mà ta đã viết để compile khai báo hàm. Hàm tiện ích này compile danh sách tham số và phần thân hàm. Sau đó nó phát sinh code để tạo một ObjClosure và để nó trên đỉnh stack. Ở runtime, VM sẽ tìm closure ở đó.

Cuối cùng là class để bind method vào. VM có thể tìm nó ở đâu? Tiếc là, khi ta đến instruction `OP_METHOD`, ta không biết nó ở đâu. Nó <span name="global">có thể</span> nằm trên stack, nếu người dùng khai báo class trong local scope. Nhưng một khai báo class ở cấp cao nhất sẽ đưa ObjClass vào bảng biến global.

<aside name="global">

Nếu Lox chỉ hỗ trợ khai báo class ở cấp cao nhất, VM có thể giả định rằng bất kỳ class nào cũng có thể được tìm thấy bằng cách tra trực tiếp từ bảng biến global. Tiếc là, vì ta hỗ trợ class cục bộ, nên ta cần xử lý cả trường hợp đó nữa.

</aside>


Đừng lo. Compiler **có** biết *tên* của class. Ta có thể lưu lại nó ngay sau khi tiêu thụ token của nó.

^code class-name (1 before, 1 after)

Và ta biết rằng sẽ không có khai báo nào khác với tên đó có thể shadow class này. Vậy nên ta làm một cách xử lý đơn giản: trước khi bắt đầu bind method, ta phát sinh bất kỳ code nào cần thiết để load class trở lại lên đỉnh stack.

^code load-class (2 before, 1 after)

Ngay trước khi compile phần thân class, ta <span name="load">gọi</span> `namedVariable()`. Hàm helper này sẽ tạo code để load một biến với tên đã cho lên stack. Sau đó ta compile các method.

<aside name="load">

Lệnh gọi `defineVariable()` ngay trước đó sẽ pop class, nên việc gọi `namedVariable()` để load nó lại lên stack nghe có vẻ hơi buồn cười. Sao không để nguyên nó trên stack ngay từ đầu? Ta **có thể** làm vậy, nhưng trong [chương tiếp theo](superclasses.html) ta sẽ chèn code giữa hai lệnh này để hỗ trợ kế thừa. Lúc đó, sẽ đơn giản hơn nếu class không “nằm chình ình” trên stack.

</aside>

Điều này có nghĩa là khi ta execute mỗi instruction `OP_METHOD`, stack sẽ có closure của method ở trên cùng và class ngay bên dưới. Khi compile xong tất cả method, ta không cần class nữa và yêu cầu VM pop nó khỏi stack.

^code pop-class (1 before, 1 after)

Gộp tất cả lại, đây là một ví dụ khai báo class để “ném” vào compiler:

```lox
class Brunch {
  bacon() {}
  eggs() {}
}
```

Với ví dụ này, đây là những gì compiler tạo ra và cách các instruction đó ảnh hưởng đến stack ở runtime:

<img src="image/methods-and-initializers/method-instructions.png" alt="Chuỗi bytecode instruction cho một khai báo class với hai method." />

Giờ việc còn lại là implement runtime cho instruction `OP_METHOD` mới này.

### Execute khai báo method

Đầu tiên, ta định nghĩa opcode.

^code method-op (1 before, 1 after)

Ta disassemble nó giống như các instruction khác có toán hạng là string constant.

^code disassemble-method (2 before, 1 after)

Và bên trong interpreter, ta thêm một case mới.

^code interpret-method (1 before, 1 after)

Tại đây, ta đọc tên method từ constant table và truyền nó vào đây:

^code define-method

Closure của method nằm trên đỉnh stack, ngay phía trên class mà nó sẽ được bind vào. Ta đọc hai slot stack đó và lưu closure vào bảng method của class. Sau đó ta pop closure vì đã xong việc với nó.

Lưu ý rằng ta không thực hiện bất kỳ runtime type checking nào với closure hay object class. Lời gọi `AS_CLASS()` là an toàn vì chính compiler đã tạo ra code khiến class nằm ở slot stack đó. VM <span name="verify">tin tưởng</span> compiler của chính nó.

<aside name="verify">

VM tin rằng các instruction nó execute là hợp lệ vì *cách duy nhất* để đưa code vào bytecode interpreter là thông qua compiler của clox. Nhiều bytecode VM khác, như JVM và CPython, hỗ trợ execute bytecode được compile riêng. Điều đó dẫn đến một câu chuyện bảo mật khác: bytecode được tạo ra với mục đích xấu có thể làm crash VM hoặc tệ hơn.

Để ngăn điều đó, JVM thực hiện một bước xác minh bytecode trước khi execute bất kỳ code nào được load. CPython thì nói rằng việc đảm bảo bytecode an toàn là trách nhiệm của người dùng.

</aside>

Sau khi chuỗi instruction `OP_METHOD` hoàn tất và `OP_POP` đã pop class, ta sẽ có một class với bảng method được “lấp đầy” gọn gàng, sẵn sàng hoạt động. Bước tiếp theo là lấy các method đó ra và sử dụng chúng.

## Tham chiếu Method

Phần lớn thời gian, method được truy cập và gọi ngay lập tức, dẫn đến cú pháp quen thuộc này:

```lox
instance.method(argument);
```

Nhưng hãy nhớ, trong Lox và một số ngôn ngữ khác, hai bước này là riêng biệt và có thể tách rời.

```lox
var closure = instance.method;
closure(argument);
```

Vì người dùng *có thể* tách riêng các thao tác này, ta phải implement chúng riêng biệt. Bước đầu tiên là dùng cú pháp property có dấu chấm hiện có để truy cập một method được định nghĩa trong class của instance. Việc này sẽ trả về một loại object nào đó mà người dùng có thể gọi như một hàm.

Cách tiếp cận hiển nhiên là tra method trong bảng method của class và trả về ObjClosure tương ứng với tên đó. Nhưng ta cũng cần nhớ rằng khi bạn truy cập một method, `this` sẽ được bind tới instance mà method được truy cập từ đó. Đây là ví dụ từ [khi ta thêm method vào jlox](classes.html#methods-on-classes):


```lox
class Person {
  sayName() {
    print this.name;
  }
}

var jane = Person();
jane.name = "Jane";

var method = jane.sayName;
method(); // ?
```

Đoạn code này sẽ in ra `"Jane"`, nên object được trả về bởi `.sayName` bằng cách nào đó cần phải “nhớ” instance mà nó được truy cập từ đó, để khi được gọi sau này nó vẫn biết. Trong jlox, ta hiện thực phần “ghi nhớ” này bằng cách tận dụng class Environment cấp phát trên heap sẵn có của interpreter, vốn xử lý toàn bộ việc lưu trữ biến.

Bytecode VM của ta có kiến trúc lưu trữ trạng thái phức tạp hơn. [Biến local và temporary](local-variables.html#representing-local-variables) nằm trên stack, [biến global](global-variables.html#variable-declarations) nằm trong hash table, và biến trong closure dùng [upvalue](closures.html#upvalues). Điều này đòi hỏi một giải pháp phức tạp hơn để theo dõi receiver của method trong clox, và cần một kiểu runtime mới.

### Bound method

Khi người dùng execute một thao tác truy cập method, ta sẽ tìm closure của method đó và bọc nó trong một object <span name="bound">“bound method”</span> mới, object này sẽ theo dõi instance mà method được truy cập từ đó. Bound object này có thể được gọi sau đó như một hàm. Khi được invoke, VM sẽ “dàn xếp” để `this` trỏ tới receiver bên trong phần thân method.

<aside name="bound">

Tôi lấy tên “bound method” từ CPython. Python hoạt động tương tự Lox ở điểm này, và tôi đã lấy cảm hứng từ cách hiện thực của nó.

</aside>

Đây là kiểu object mới:

^code obj-bound-method (2 before, 1 after)

Nó gói cả receiver và method closure lại với nhau. Kiểu của receiver là Value, dù method chỉ có thể được gọi trên ObjInstance. Vì VM vốn không quan tâm receiver thuộc loại gì, nên dùng Value giúp ta không phải chuyển đổi con trỏ về Value mỗi khi truyền nó cho các hàm tổng quát hơn.

Struct mới này kéo theo phần “boilerplate” quen thuộc mà giờ bạn đã quá rành. Một case mới trong enum loại object:

^code obj-type-bound-method (1 before, 1 after)

Một macro để kiểm tra kiểu của value:

^code is-bound-method (2 before, 1 after)

Một macro khác để cast value sang con trỏ ObjBoundMethod:

^code as-bound-method (2 before, 1 after)

Một hàm để tạo ObjBoundMethod mới:

^code new-bound-method-h (2 before, 1 after)

Và phần hiện thực của hàm đó:

^code new-bound-method

Hàm “constructor” này đơn giản chỉ lưu closure và receiver được truyền vào. Khi bound method không còn cần nữa, ta free nó.

^code free-bound-method (1 before, 1 after)

Bound method có một vài tham chiếu, nhưng nó không *sở hữu* chúng, nên nó chỉ free chính nó. Tuy nhiên, các tham chiếu đó vẫn được trace bởi garbage collector.

^code blacken-bound-method (1 before, 1 after)

Điều này <span name="trace">đảm bảo</span> rằng một handle tới method sẽ giữ receiver lại trong bộ nhớ để `this` vẫn có thể tìm thấy object khi bạn invoke handle đó sau này. Ta cũng trace cả method closure.

<aside name="trace">

Việc trace method closure thực ra không thật sự cần thiết. Receiver là một ObjInstance, nó có con trỏ tới ObjClass của nó, và ObjClass này có bảng chứa tất cả method. Nhưng tôi vẫn cảm thấy hơi “lấn cấn” nếu để ObjBoundMethod phụ thuộc vào điều đó.

</aside>

Thao tác cuối cùng mà mọi object hỗ trợ là in ra.

^code print-bound-method (1 before, 1 after)

Bound method được in ra giống hệt như function. Từ góc nhìn của người dùng, bound method *là* một function — một object mà họ có thể gọi. Ta không để lộ rằng VM hiện thực bound method bằng một kiểu object khác.

<aside name="party">

<img src="image/methods-and-initializers/party-hat.png" alt="Một chiếc mũ tiệc." />

</aside>

Hãy đội mũ <span name="party">party</span> lên vì ta vừa đạt một cột mốc nhỏ: ObjBoundMethod là kiểu runtime cuối cùng cần thêm vào clox. Bạn vừa viết xong macro `IS_` và `AS_` cuối cùng. Chỉ còn vài chương nữa là hết sách, và ta đang tiến rất gần tới một VM hoàn chỉnh.

### Truy cập method

Hãy để kiểu object mới của chúng ta bắt đầu “ra tay”. Method được truy cập bằng cú pháp property có dấu chấm giống hệt như ta đã implement ở chương trước. Compiler đã parse đúng các biểu thức và phát sinh instruction `OP_GET_PROPERTY` cho chúng. Việc duy nhất ta cần thay đổi là ở runtime.

Khi một instruction truy cập property được execute, instance sẽ nằm trên đỉnh stack. Nhiệm vụ của instruction là tìm một field hoặc method với tên đã cho và thay thế đỉnh stack bằng property vừa truy cập.

Interpreter đã xử lý field, nên ta chỉ cần mở rộng case `OP_GET_PROPERTY` với một phần mới.

^code get-method (5 before, 1 after)

Ta chèn đoạn này sau phần code tìm field trên receiver instance. Field được ưu tiên hơn và sẽ shadow method, nên ta tìm field trước. Nếu instance không có field với tên property đã cho, thì tên đó có thể là một method.

Ta lấy class của instance và truyền nó vào helper mới `bindMethod()`. Nếu hàm này tìm thấy method, nó sẽ đặt method lên stack và trả về `true`. Ngược lại, nó trả về `false` để báo rằng không tìm thấy method với tên đó. Vì tên này cũng không phải là field, điều đó có nghĩa là ta gặp lỗi runtime và interpreter sẽ dừng.

Đây là phần “ngon”:

^code bind-method

Đầu tiên, ta tìm method với tên đã cho trong bảng method của class. Nếu không tìm thấy, ta báo lỗi runtime và thoát. Nếu tìm thấy, ta lấy method đó và bọc nó trong một ObjBoundMethod mới. Ta lấy receiver từ vị trí của nó trên đỉnh stack. Cuối cùng, ta pop instance và thay đỉnh stack bằng bound method.

Ví dụ:

```lox
class Brunch {
  eggs() {}
}

var brunch = Brunch();
var eggs = brunch.eggs;
```

Đây là những gì xảy ra khi VM execute lời gọi `bindMethod()` cho biểu thức `brunch.eggs`:

<img src="image/methods-and-initializers/bind-method.png" alt="Những thay đổi trên stack do bindMethod() gây ra." />

Có khá nhiều “máy móc” chạy ngầm bên dưới, nhưng từ góc nhìn của người dùng, họ chỉ đơn giản nhận được một hàm mà họ có thể gọi.

### Gọi method

Người dùng có thể khai báo method trong class, truy cập chúng từ instance, và đưa bound method lên stack. Nhưng họ vẫn chưa thể <span name="do">*làm*</span> gì hữu ích với các object bound method đó. Thao tác còn thiếu là gọi chúng. Việc gọi được implement trong `callValue()`, nên ta thêm một case ở đó cho kiểu object mới này.

<aside name="do">

Bound method *là* một giá trị hạng nhất, nên chúng có thể được lưu trong biến, truyền vào hàm, và làm những việc “giống giá trị” khác.

</aside>

^code call-bound-method (1 before, 1 after)

Ta lấy lại closure “thô” từ ObjBoundMethod và dùng helper `call()` sẵn có để bắt đầu invoke closure đó bằng cách push một CallFrame cho nó lên call stack. Chỉ cần vậy là ta có thể chạy chương trình Lox này:

```lox
class Scone {
  topping(first, second) {
    print "scone with " + first + " and " + second;
  }
}

var scone = Scone();
scone.topping("berries", "cream");
```

Vậy là ta đã có ba bước lớn: khai báo, truy cập, và invoke method. Nhưng vẫn còn thiếu một thứ. Ta đã mất công bọc closure của method trong một object bind receiver, nhưng khi invoke method, ta lại không hề dùng đến receiver đó.

## This

Lý do bound method cần giữ lại receiver là để có thể truy cập nó bên trong phần thân method. Lox cung cấp receiver của method thông qua biểu thức `this`. Đã đến lúc thêm một chút cú pháp mới. Lexer đã coi `this` là một loại token đặc biệt, nên bước đầu tiên là nối token đó vào parse table.

^code table-this (1 before, 1 after)

<aside name="this">

Dấu gạch dưới ở cuối tên hàm parser là vì `this` là một từ khóa dành riêng trong C++, và ta hỗ trợ compile clox dưới dạng C++.

</aside>

Khi parser gặp `this` ở vị trí prefix, nó sẽ chuyển sang một hàm parser mới.

^code this

Ta sẽ áp dụng cùng kỹ thuật hiện thực `this` trong clox như đã dùng trong jlox. Ta coi `this` như một biến local có phạm vi từ vựng (lexically scoped) với giá trị được khởi tạo “một cách kỳ diệu”. Compile nó như một biến local giúp ta có được nhiều hành vi miễn phí. Đặc biệt, closure bên trong một method mà tham chiếu `this` sẽ hoạt động đúng và capture receiver vào một upvalue.

Khi hàm parser này được gọi, token `this` vừa được tiêu thụ và được lưu dưới dạng token trước đó. Ta gọi hàm `variable()` sẵn có, hàm này compile biểu thức identifier như các truy cập biến. Nó nhận một tham số Boolean duy nhất để cho biết compiler có nên tìm toán tử `=` theo sau và parse một setter hay không. Bạn không thể gán cho `this`, nên ta truyền `false` để không cho phép điều đó.

Hàm `variable()` không quan tâm việc `this` có token type riêng và không phải là identifier. Nó vui vẻ coi lexeme `"this"` như một tên biến và tra cứu nó bằng cơ chế resolve scope sẵn có. Hiện tại, việc tra cứu đó sẽ thất bại vì ta chưa bao giờ khai báo một biến tên `"this"`. Đã đến lúc nghĩ xem receiver nên nằm ở đâu trong bộ nhớ.

Ít nhất là cho đến khi bị capture bởi closure, clox lưu mọi biến local trên stack của VM. Compiler theo dõi slot nào trong “cửa sổ” stack của hàm thuộc về biến local nào. Nếu bạn còn nhớ, compiler dành riêng slot 0 trên stack bằng cách khai báo một biến local có tên là chuỗi rỗng.

Với lời gọi hàm, slot đó sẽ chứa hàm được gọi. Vì slot này không có tên, phần thân hàm sẽ không bao giờ truy cập nó. Bạn có thể đoán được hướng đi rồi đấy. Với *lời gọi method*, ta có thể tái sử dụng slot đó để lưu receiver. Slot 0 sẽ chứa instance mà `this` được bind tới. Để compile biểu thức `this`, compiler chỉ cần gán đúng tên cho biến local đó.

^code slot-zero (1 before, 1 after)

Ta chỉ muốn làm điều này cho method. Khai báo hàm không có `this`. Và thực tế, chúng *không được* khai báo biến tên `"this"`, để nếu bạn viết một biểu thức `this` bên trong một khai báo hàm nằm trong một method, thì `this` sẽ resolve đúng tới receiver của method bên ngoài.

```lox
class Nested {
  method() {
    fun function() {
      print this;
    }

    function();
  }
}

Nested().method();
```

Chương trình này sẽ in ra `"Nested instance"`. Để quyết định tên gán cho slot 0 local, compiler cần biết nó đang compile một khai báo hàm hay method, nên ta thêm một case mới vào enum FunctionType để phân biệt method.

^code method-type-enum (1 before, 1 after)

Khi compile một method, ta dùng loại đó.

^code method-type (2 before, 1 after)

Giờ ta có thể compile đúng các tham chiếu tới biến đặc biệt `"this"`, và compiler sẽ phát sinh instruction `OP_GET_LOCAL` phù hợp để truy cập nó. Closure thậm chí có thể capture `this` và lưu receiver vào upvalue. Khá tuyệt.

Ngoại trừ việc ở runtime, receiver thực ra chưa *nằm* trong slot 0. Interpreter vẫn chưa thực hiện phần việc của nó. Đây là cách sửa:

^code store-receiver (2 before, 2 after)

Khi một method được gọi, đỉnh stack chứa tất cả các đối số, và ngay bên dưới chúng là closure của method được gọi. Đó chính là vị trí slot 0 trong CallFrame mới. Dòng code này chèn receiver vào slot đó. Ví dụ, với lời gọi method như sau:

```lox
scone.topping("berries", "cream");
```

Ta tính slot để lưu receiver như sau:

<img src="image/methods-and-initializers/closure-slot.png" alt="Bỏ qua các slot đối số trên stack để tìm slot chứa closure." />

Phần `-argCount` bỏ qua các đối số và `- 1` điều chỉnh vì `stackTop` trỏ ngay *sau* slot stack cuối cùng được dùng.

### Sử dụng sai `this`

VM của chúng ta giờ đã hỗ trợ người dùng sử dụng `this` *đúng cách*, nhưng ta cũng cần đảm bảo nó xử lý đúng khi người dùng *dùng sai* `this`. Lox quy định rằng nếu một biểu thức `this` xuất hiện bên ngoài phần thân của một method thì đó là lỗi compile. Hai trường hợp sai sau đây cần được compiler phát hiện:

```lox
print this; // Ở top level.

fun notMethod() {
  print this; // Bên trong một hàm.
}
```

Vậy compiler biết mình đang ở bên trong một method bằng cách nào? Câu trả lời hiển nhiên là nhìn vào `FunctionType` của `Compiler` hiện tại. Ta vừa thêm một case enum để xử lý method đặc biệt. Tuy nhiên, cách này sẽ không xử lý đúng đoạn code như ví dụ trước đó, nơi bạn đang ở bên trong một hàm mà bản thân nó lại được lồng bên trong một method.

Ta có thể thử resolve `"this"` rồi báo lỗi nếu nó không được tìm thấy trong bất kỳ scope từ vựng bao quanh nào. Cách này sẽ hoạt động, nhưng sẽ yêu cầu ta phải sắp xếp lại khá nhiều code, vì hiện tại code resolve biến ngầm định coi đó là truy cập global nếu không tìm thấy khai báo.

Trong chương tiếp theo, ta sẽ cần thông tin về class bao ngoài gần nhất. Nếu có thông tin đó, ta có thể dùng nó ở đây để xác định xem mình có đang ở trong một method hay không. Vậy nên tốt nhất là chuẩn bị sẵn cơ chế này ngay bây giờ để sau này đỡ vất vả.

^code current-class (1 before, 2 after)

Biến module này trỏ tới một struct đại diện cho class trong cùng hiện tại đang được compile. Kiểu mới này trông như sau:

^code class-compiler-struct (1 before, 2 after)

Hiện tại, ta chỉ lưu một con trỏ tới `ClassCompiler` của class bao ngoài, nếu có. Việc lồng một khai báo class bên trong một method của class khác là điều hiếm gặp, nhưng Lox vẫn hỗ trợ. Giống như struct `Compiler`, điều này có nghĩa là `ClassCompiler` tạo thành một danh sách liên kết từ class trong cùng hiện tại đang compile ra tới tất cả các class bao ngoài.

Nếu ta không ở trong bất kỳ khai báo class nào, biến module `currentClass` sẽ là `NULL`. Khi compiler bắt đầu compile một class, nó sẽ push một `ClassCompiler` mới vào “ngăn xếp” liên kết ngầm này.

^code create-class-compiler (2 before, 1 after)

Vùng nhớ cho struct `ClassCompiler` nằm ngay trên stack của C, một khả năng tiện lợi mà ta có được nhờ viết compiler theo kiểu recursive descent. Khi kết thúc phần thân class, ta pop compiler đó ra khỏi stack và khôi phục lại class bao ngoài.

^code pop-enclosing (1 before, 1 after)

Khi phần thân của class ngoài cùng kết thúc, `enclosing` sẽ là `NULL`, nên lúc này `currentClass` được đặt lại thành `NULL`. Do đó, để biết ta có đang ở trong một class — và vì thế là trong một method — hay không, ta chỉ cần kiểm tra biến module này.

^code this-outside-class (1 before, 1 after)

Với cơ chế này, `this` bên ngoài một class sẽ bị cấm đúng như quy định. Giờ thì method của ta thực sự mang đúng nghĩa *method* trong lập trình hướng đối tượng. Việc truy cập receiver cho phép chúng tác động đến instance mà bạn gọi method trên đó. Chúng ta đang tiến triển tốt!

## Instance Initializer

Lý do các ngôn ngữ hướng đối tượng gắn kết state và behavior — một trong những nguyên lý cốt lõi của mô hình này — là để đảm bảo object luôn ở trạng thái hợp lệ và có ý nghĩa. Khi cách duy nhất để tác động đến state của object là <span name="through">thông qua</span> các method của nó, các method có thể đảm bảo không có gì sai sót. Nhưng điều đó giả định rằng object đã *sẵn* ở trạng thái đúng. Vậy khi nó vừa được tạo ra thì sao?

<aside name="through">

Tất nhiên, Lox cho phép code bên ngoài truy cập và sửa đổi trực tiếp các field của instance mà không cần thông qua method. Điều này khác với Ruby và Smalltalk, vốn đóng gói hoàn toàn state bên trong object. Ngôn ngữ script “đồ chơi” của chúng ta, tiếc là, không nguyên tắc đến vậy.

</aside>

Các ngôn ngữ hướng đối tượng đảm bảo object mới được tạo ra được thiết lập đúng thông qua constructor, vừa tạo ra một instance mới vừa khởi tạo state của nó. Trong Lox, runtime sẽ cấp phát các instance thô mới, và một class có thể khai báo một initializer để thiết lập các field. Initializer hoạt động gần giống như method bình thường, nhưng có một vài điểm khác:

1.  Runtime sẽ tự động invoke method initializer bất cứ khi nào một instance của class được tạo.

2.  Caller tạo instance luôn nhận lại instance đó <span name="return">sau</span> khi initializer kết thúc, bất kể bản thân hàm initializer trả về gì. Method initializer không cần phải `return this` một cách tường minh.

3.  Thực tế, initializer *bị cấm* trả về bất kỳ giá trị nào vì giá trị đó sẽ không bao giờ được dùng đến.

<aside name="return">

Giống như thể initializer được ngầm bọc trong một đoạn code như sau:

```lox
fun create(klass) {
  var obj = newInstance(klass);
  obj.init();
  return obj;
}
```

Hãy chú ý rằng giá trị trả về từ `init()` bị bỏ qua.

</aside>

Giờ khi ta đã hỗ trợ method, để thêm initializer, ta chỉ cần implement ba quy tắc đặc biệt này. Ta sẽ làm theo thứ tự.

### Gọi initializer

Đầu tiên, tự động gọi `init()` trên các instance mới:

^code call-init (1 before, 1 after)

Sau khi runtime cấp phát instance mới, ta tìm method `init()` trong class. Nếu tìm thấy, ta bắt đầu gọi nó. Việc này sẽ push một CallFrame mới cho closure của initializer. Giả sử ta chạy chương trình này:

```lox
class Brunch {
  init(food, drink) {}
}

Brunch("eggs", "coffee");
```

Khi VM execute lời gọi `Brunch()`, quá trình diễn ra như sau:

<img src="image/methods-and-initializers/init-call-frame.png" alt="Các cửa sổ stack được căn chỉnh cho lời gọi Brunch() và method init() tương ứng mà nó chuyển tiếp đến." />

Bất kỳ đối số nào được truyền vào class khi ta gọi nó vẫn nằm trên stack phía trên instance. CallFrame mới cho method `init()` sẽ dùng chung cửa sổ stack đó, nên các đối số này sẽ được chuyển tiếp ngầm định tới initializer.

Lox không bắt buộc một class phải định nghĩa initializer. Nếu bỏ qua, runtime đơn giản trả về instance mới chưa được khởi tạo. Tuy nhiên, nếu không có method `init()`, thì việc truyền đối số vào class khi tạo instance là vô nghĩa. Ta coi đó là lỗi.

^code no-init-arity-error (1 before, 1 after)

Khi class *có* cung cấp initializer, ta cũng cần đảm bảo số lượng đối số truyền vào khớp với arity của initializer. May mắn là helper `call()` đã làm điều đó cho ta.

Để gọi initializer, runtime tra cứu method `init()` theo tên. Ta muốn việc này nhanh vì nó diễn ra mỗi khi một instance được tạo. Điều đó có nghĩa là nên tận dụng string interning mà ta đã implement. Để làm vậy, VM tạo một ObjString cho `"init"` và tái sử dụng nó. String này được lưu ngay trong struct VM.

^code vm-init-string (1 before, 1 after)

Ta tạo và intern string này khi VM khởi động.

^code init-init-string (1 before, 2 after)

Ta muốn nó tồn tại lâu dài, nên GC sẽ coi nó là một root.

^code mark-init-string (1 before, 1 after)

Hãy nhìn kỹ. Thấy lỗi tiềm ẩn nào không? Không à? Đây là một lỗi tinh vi. Garbage collector giờ sẽ đọc `vm.initString`. Trường này được khởi tạo từ kết quả của lời gọi `copyString()`. Nhưng việc copy string sẽ cấp phát bộ nhớ, có thể kích hoạt GC. Nếu collector chạy đúng vào thời điểm xấu, nó sẽ đọc `vm.initString` trước khi trường này được khởi tạo. Vì vậy, trước tiên ta gán giá trị 0 cho trường này.

^code null-init-string (2 before, 2 after)

Ta xóa con trỏ này khi VM tắt vì dòng tiếp theo sẽ free nó.

^code clear-init-string (1 before, 1 after)

OK, vậy là ta đã có thể gọi initializer.

### Giá trị trả về của initializer

Bước tiếp theo là đảm bảo rằng việc tạo một instance của class có initializer luôn trả về instance mới, chứ không phải `nil` hay bất kỳ giá trị nào mà phần thân initializer trả về. Hiện tại, nếu một class định nghĩa initializer, thì khi một instance được tạo, VM sẽ push một lời gọi tới initializer đó lên CallFrame stack. Sau đó nó cứ tiếp tục chạy.

Lời gọi của người dùng tới class để tạo instance sẽ hoàn tất khi method initializer trả về, và sẽ để lại trên stack bất kỳ giá trị nào mà initializer đặt ở đó. Điều này có nghĩa là trừ khi người dùng cẩn thận đặt `return this;` ở cuối initializer, sẽ không có instance nào được trả về. Không hữu ích lắm.

Để khắc phục, bất cứ khi nào front end compile một method initializer, nó sẽ phát sinh bytecode khác ở cuối phần thân để trả về `this` từ method thay vì `nil` ngầm định mà hầu hết các hàm trả về. Để *làm được* điều đó, compiler cần biết khi nào nó đang compile một initializer. Ta phát hiện điều này bằng cách kiểm tra xem tên method đang compile có phải `"init"` hay không.

^code initializer-name (1 before, 1 after)

Ta định nghĩa một loại function mới để phân biệt initializer với các method khác.

^code initializer-type-enum (1 before, 1 after)

Bất cứ khi nào compiler phát sinh lệnh return ngầm ở cuối phần thân, ta kiểm tra loại để quyết định có chèn hành vi đặc biệt cho initializer hay không.

^code return-this (1 before, 1 after)

Trong initializer, thay vì push `nil` lên stack trước khi return, ta load slot 0, nơi chứa instance. Hàm `emitReturn()` này cũng được gọi khi compile một câu lệnh `return` không có giá trị, nên nó cũng xử lý đúng các trường hợp người dùng return sớm bên trong initializer.

### Lệnh `return` không hợp lệ trong initializer

Bước cuối cùng, mục cuối trong danh sách các tính năng đặc biệt của initializer, là biến việc cố gắng trả về bất kỳ thứ gì *khác* thành lỗi. Giờ khi compiler đã theo dõi loại method, việc này trở nên đơn giản.

^code return-from-init (3 before, 1 after)

Ta báo lỗi nếu một câu lệnh `return` trong initializer có giá trị trả về. Ta vẫn tiếp tục compile giá trị đó sau đó để compiler không bị nhầm lẫn bởi biểu thức theo sau và báo một loạt lỗi dây chuyền.

Ngoại trừ phần kế thừa, mà ta sẽ bàn tới [sớm thôi](super), giờ ta đã có một hệ thống class khá đầy đủ tính năng hoạt động trong clox.

```lox
class CoffeeMaker {
  init(coffee) {
    this.coffee = coffee;
  }

  brew() {
    print "Enjoy your cup of " + this.coffee;

    // Không tái sử dụng bã cà phê nhé!
    this.coffee = nil;
  }
}

var maker = CoffeeMaker("coffee and chicory");
maker.brew();
```

Khá “xịn” đối với một chương trình C có thể vừa trên một chiếc <span name="floppy">đĩa mềm</span> đời cũ.

<aside name="floppy">

Tôi biết rằng “đĩa mềm” có thể không còn là một đơn vị so sánh hữu ích cho thế hệ lập trình viên hiện nay. Có lẽ tôi nên nói “vài tweet” hay gì đó tương tự.

</aside>

## Lời gọi được tối ưu hóa

VM của ta hiện đã implement đúng ngữ nghĩa của ngôn ngữ cho lời gọi method và initializer. Ta có thể dừng ở đây. Nhưng lý do chính khiến ta xây dựng một bản hiện thực thứ hai của Lox từ đầu là để execute nhanh hơn interpreter Java cũ. Hiện tại, ngay cả trong clox, lời gọi method vẫn chậm.

Ngữ nghĩa của Lox định nghĩa một lời gọi method gồm hai thao tác — truy cập method và sau đó gọi kết quả. VM của ta phải hỗ trợ chúng như hai thao tác riêng biệt vì người dùng *có thể* tách chúng ra. Bạn có thể truy cập một method mà không gọi nó, rồi invoke bound method đó sau. Không có gì ta đã implement đến giờ là thừa thãi.

Nhưng *luôn* execute chúng như hai thao tác riêng biệt lại có chi phí đáng kể. Mỗi lần một chương trình Lox truy cập và gọi một method, runtime heap sẽ cấp phát một ObjBoundMethod mới, khởi tạo các field của nó, rồi ngay lập tức lấy chúng ra. Sau đó, GC phải tốn thời gian giải phóng tất cả các bound method ngắn ngủi đó.

Phần lớn thời gian, một chương trình Lox truy cập một method và ngay lập tức gọi nó. Bound method được tạo ra bởi một bytecode instruction và bị tiêu thụ ngay bởi instruction tiếp theo. Thực tế, nó diễn ra nhanh đến mức compiler thậm chí có thể *thấy* rõ điều đó — một truy cập property bằng dấu chấm theo sau bởi dấu ngoặc mở rất có khả năng là một lời gọi method.

Vì ta có thể nhận diện cặp thao tác này ngay tại compile time, ta có cơ hội phát sinh một instruction <span name="super">mới, đặc biệt</span> để thực hiện một lời gọi method được tối ưu hóa.

Ta bắt đầu trong hàm compile các biểu thức property có dấu chấm.

<aside name="super" class="bottom">

Nếu bạn dành đủ thời gian quan sát bytecode VM chạy, bạn sẽ nhận thấy nó thường execute cùng một chuỗi bytecode instruction hết lần này đến lần khác. Một kỹ thuật tối ưu hóa kinh điển là định nghĩa một instruction đơn mới gọi là **superinstruction** để hợp nhất chúng thành một instruction duy nhất với hành vi giống hệt toàn bộ chuỗi.

Một trong những nguyên nhân lớn nhất gây hao phí hiệu năng trong bytecode interpreter là chi phí giải mã và phân phối từng instruction. Hợp nhất nhiều instruction thành một sẽ loại bỏ được phần nào chi phí đó.

Thách thức là xác định *chuỗi* instruction nào đủ phổ biến để hưởng lợi từ tối ưu hóa này. Mỗi superinstruction mới sẽ chiếm một opcode riêng và số lượng opcode là hữu hạn. Thêm quá nhiều, bạn sẽ cần một cách mã hóa opcode lớn hơn, điều này sẽ làm tăng kích thước code và khiến việc giải mã *tất cả* instruction chậm hơn.

</aside>

^code parse-call (3 before, 1 after)

Sau khi compiler đã parse xong tên property, ta kiểm tra xem có dấu ngoặc tròn mở hay không. Nếu có, ta chuyển sang một nhánh code mới. Ở đó, ta compile danh sách đối số giống hệt như khi compile một call expression. Sau đó, ta phát sinh một instruction `OP_INVOKE` mới duy nhất. Nó nhận hai toán hạng:

1.  Chỉ số của tên property trong constant table.

2.  Số lượng đối số được truyền vào method.

Nói cách khác, instruction duy nhất này kết hợp các toán hạng của `OP_GET_PROPERTY` và `OP_CALL` mà nó thay thế, theo đúng thứ tự đó. Nó thực sự là sự hợp nhất của hai instruction này. Hãy định nghĩa nó.

^code invoke-op (1 before, 1 after)

Và thêm nó vào disassembler:

^code disassemble-invoke (2 before, 1 after)

Đây là một định dạng instruction mới, đặc biệt, nên cần một chút logic disassembly tùy chỉnh.

^code invoke-instruction

Ta đọc hai toán hạng và sau đó in ra cả tên method lẫn số lượng đối số. Bên trong vòng lặp dispatch bytecode của interpreter mới là nơi mọi thứ thực sự bắt đầu.

^code interpret-invoke (1 before, 1 after)

Phần lớn công việc diễn ra trong `invoke()`, mà ta sẽ bàn tới ngay. Ở đây, ta tra cứu tên method từ toán hạng đầu tiên rồi đọc toán hạng số lượng đối số. Sau đó, ta chuyển cho `invoke()` để xử lý phần nặng. Hàm này trả về `true` nếu lời gọi thành công. Như thường lệ, trả về `false` nghĩa là đã xảy ra lỗi runtime. Ta kiểm tra điều đó ở đây và dừng interpreter nếu có sự cố.

Cuối cùng, giả sử lời gọi thành công, sẽ có một CallFrame mới trên stack, nên ta làm mới bản sao cache của frame hiện tại trong `frame`.

Phần thú vị diễn ra ở đây:

^code invoke

Đầu tiên, ta lấy receiver từ stack. Các đối số truyền vào method nằm phía trên nó trên stack, nên ta peek xuống bấy nhiêu slot. Sau đó, chỉ đơn giản là cast object đó thành instance và invoke method trên nó.

Điều này giả định rằng object *là* một instance. Giống như với instruction `OP_GET_PROPERTY`, ta cũng cần xử lý trường hợp người dùng cố gọi method trên một giá trị sai kiểu.

^code invoke-check-type (1 before, 1 after)

<span name="helper">Đây</span> là một lỗi runtime, nên ta báo lỗi và thoát. Nếu không, ta lấy class của instance và nhảy sang hàm tiện ích mới này:

<aside name="helper">

Như bạn có thể đoán, ta tách code này thành một hàm riêng vì sẽ tái sử dụng nó sau này — trong trường hợp này là cho các lời gọi `super`.

</aside>

^code invoke-from-class

Hàm này kết hợp logic của cách VM hiện thực instruction `OP_GET_PROPERTY` và `OP_CALL`, theo đúng thứ tự đó. Đầu tiên, ta tra method theo tên trong bảng method của class. Nếu không tìm thấy, ta báo lỗi runtime và thoát.

Nếu tìm thấy, ta lấy closure của method và push một lời gọi tới nó lên CallFrame stack. Ta không cần cấp phát trên heap và khởi tạo một ObjBoundMethod. Thực tế, ta thậm chí không cần phải <span name="juggle">xoay</span> gì trên stack. Receiver và các đối số của method đã ở đúng vị trí cần thiết.

<aside name="juggle">

Đây là lý do chính *tại sao* ta dùng slot 0 trên stack để lưu receiver — đó là cách caller đã tổ chức stack cho một lời gọi method. Một calling convention hiệu quả là phần quan trọng trong câu chuyện hiệu năng của một bytecode VM.

</aside>

Nếu bạn khởi động VM và chạy một chương trình nhỏ gọi method ngay bây giờ, bạn sẽ thấy hành vi *y hệt* như trước. Nhưng, nếu ta làm đúng, *hiệu năng* sẽ được cải thiện đáng kể. Tôi đã viết một microbenchmark nhỏ thực hiện một lô 10.000 lời gọi method. Sau đó, nó kiểm tra xem có thể execute bao nhiêu lô như vậy trong 10 giây. Trên máy của tôi, không có instruction `OP_INVOKE` mới, nó chạy được 1.089 lô. Với tối ưu hóa mới này, nó hoàn thành 8.324 lô trong cùng thời gian. Đó là *nhanh hơn 7,6 lần*, một cải thiện khổng lồ khi nói đến tối ưu hóa ngôn ngữ lập trình.

<span name="pat"></span>

<aside name="pat">

Ta cũng đừng tự vỗ lưng *quá* mạnh. Cải thiện hiệu năng này là so với chính hiện thực lời gọi method chưa tối ưu của ta, vốn khá chậm. Việc cấp phát trên heap cho *mỗi* lời gọi method chắc chắn không thể thắng trong cuộc đua nào cả.

</aside>

<img src="image/methods-and-initializers/benchmark.png" alt="Bar chart comparing the two benchmark results." />

### Gọi field

Tôn chỉ cơ bản của tối ưu hóa là: *"Ngươi chớ phá vỡ tính đúng đắn."*  
<span name="monte">Người dùng</span> sẽ thích khi một hiện thực ngôn ngữ cho họ câu trả lời nhanh hơn, nhưng chỉ khi đó là câu trả lời *đúng*. Tiếc thay, phần hiện thực lời gọi method nhanh hơn của ta lại không giữ được nguyên tắc này:

```lox
class Oops {
  init() {
    fun f() {
      print "not a method";
    }

    this.field = f;
  }
}

var oops = Oops();
oops.field();
```

Dòng cuối trông giống như một lời gọi method. Compiler nghĩ vậy và cẩn thận phát sinh instruction `OP_INVOKE` cho nó. Tuy nhiên, thực tế không phải vậy. Điều đang xảy ra là một truy cập *field* trả về một hàm, rồi hàm đó được gọi. Hiện tại, thay vì execute đúng, VM của ta lại báo lỗi runtime khi không tìm thấy method tên `"field"`.

<aside name="monte">

Có những trường hợp người dùng có thể hài lòng khi một chương trình đôi khi trả về kết quả sai để đổi lấy việc chạy nhanh hơn đáng kể hoặc có giới hạn hiệu năng tốt hơn. Đây là lĩnh vực của [**thuật toán Monte Carlo**](https://en.wikipedia.org/wiki/Monte_Carlo_algorithm). Với một số trường hợp sử dụng, đây là một sự đánh đổi hợp lý.

Điều quan trọng, tuy nhiên, là người dùng *tự chọn* áp dụng một trong các thuật toán này. Chúng ta — những người hiện thực ngôn ngữ — không thể tự ý quyết định hy sinh tính đúng đắn của chương trình của họ.

</aside>

Trước đây, khi hiện thực `OP_GET_PROPERTY`, ta đã xử lý cả truy cập field và method. Để diệt bug mới này, ta cần làm điều tương tự cho `OP_INVOKE`.

^code invoke-field (1 before, 1 after)

Sửa khá đơn giản. Trước khi tra method trong class của instance, ta tìm một field có cùng tên. Nếu tìm thấy field, ta lưu nó lên stack thay cho receiver, *bên dưới* danh sách đối số. Đây chính là cách `OP_GET_PROPERTY` hoạt động, vì instruction đó được execute trước khi danh sách đối số trong ngoặc được đánh giá.

Sau đó, ta thử gọi giá trị của field đó như một callable (nếu may mắn nó là callable). Helper `callValue()` sẽ kiểm tra kiểu giá trị và gọi nó nếu phù hợp, hoặc báo lỗi runtime nếu giá trị của field không phải là kiểu callable như closure.

Chỉ cần vậy là tối ưu hóa của ta an toàn hoàn toàn. Dĩ nhiên, ta có hy sinh một chút hiệu năng. Nhưng đôi khi đó là cái giá phải trả. Thỉnh thoảng bạn sẽ thấy khó chịu vì những tối ưu hóa *có thể* làm được nếu ngôn ngữ không cho phép một vài trường hợp góc cạnh khó chịu. Nhưng, với tư cách là <span name="designer">người hiện thực</span> ngôn ngữ, ta phải chơi theo luật đã có.

<aside name="designer">

Còn với tư cách là *nhà thiết kế* ngôn ngữ, vai trò của ta rất khác. Nếu ta kiểm soát chính ngôn ngữ, đôi khi ta có thể chọn hạn chế hoặc thay đổi ngôn ngữ theo cách cho phép tối ưu hóa. Người dùng muốn ngôn ngữ biểu đạt tốt, nhưng họ cũng muốn hiện thực chạy nhanh. Đôi khi, hy sinh một chút sức mạnh để đổi lấy hiệu năng là một thiết kế ngôn ngữ tốt.

</aside>

Code ta vừa viết ở đây tuân theo một mẫu tối ưu hóa điển hình:

1.  Nhận diện một thao tác hoặc chuỗi thao tác phổ biến và quan trọng về hiệu năng. Trong trường hợp này là truy cập method rồi gọi nó.

2.  Thêm một hiện thực tối ưu cho mẫu đó. Đây chính là instruction `OP_INVOKE`.

3.  Bảo vệ code tối ưu bằng một số logic điều kiện để xác nhận mẫu thực sự áp dụng. Nếu đúng, ở lại “đường nhanh”. Nếu không, quay về hành vi chậm hơn nhưng chắc chắn hơn. Ở đây, điều đó nghĩa là kiểm tra xem ta thực sự đang gọi method chứ không phải truy cập field.

Khi công việc với ngôn ngữ của bạn chuyển từ việc khiến nó *chạy được* sang khiến nó *chạy nhanh hơn*, bạn sẽ dành ngày càng nhiều thời gian tìm kiếm những mẫu như thế này và thêm các tối ưu hóa có điều kiện cho chúng. Các kỹ sư VM toàn thời gian dành phần lớn sự nghiệp của họ trong vòng lặp này.

Nhưng ta có thể tạm dừng ở đây. Với điều này, clox giờ đã hỗ trợ hầu hết các tính năng của một ngôn ngữ lập trình hướng đối tượng, và với hiệu năng đáng nể.

<div class="challenges">

## Thử thách

1.  Việc tra cứu hash table để tìm method `init()` của class là O(1), nhưng vẫn khá chậm. Hãy hiện thực một cách nhanh hơn. Viết benchmark và đo sự khác biệt về hiệu năng.

2.  Trong một ngôn ngữ kiểu động như Lox, một callsite có thể gọi nhiều method khác nhau trên nhiều class khác nhau trong suốt quá trình chạy chương trình. Tuy nhiên, trên thực tế, phần lớn thời gian một callsite sẽ gọi đúng cùng một method trên cùng một class trong suốt thời gian chạy. Hầu hết các lời gọi thực ra không đa hình, dù ngôn ngữ cho phép.

    Các hiện thực ngôn ngữ tiên tiến tối ưu hóa dựa trên quan sát này như thế nào?

3.  Khi interpret một instruction `OP_INVOKE`, VM phải thực hiện hai lần tra cứu hash table. Đầu tiên, nó tìm một field có thể shadow method, và chỉ khi thất bại mới tìm method. Việc kiểm tra đầu tiên này hiếm khi hữu ích — hầu hết field không chứa hàm. Nhưng nó là *bắt buộc* vì ngôn ngữ quy định field và method được truy cập bằng cùng cú pháp, và field sẽ shadow method.

    Đây là một *lựa chọn* thiết kế ngôn ngữ ảnh hưởng đến hiệu năng của hiện thực. Liệu đó có phải là lựa chọn đúng? Nếu Lox là ngôn ngữ của bạn, bạn sẽ làm gì?

</div>

<div class="design-note">

## Ghi chú thiết kế: Ngân sách “độ mới lạ” (Novelty Budget)

Tôi vẫn nhớ lần đầu tiên mình viết một chương trình BASIC bé xíu trên chiếc TRS-80 và khiến máy tính làm một điều mà trước đó nó chưa từng làm. Cảm giác như có siêu năng lực vậy. Lần đầu tiên tôi chắp vá đủ một parser và interpreter để có thể viết một chương trình nhỏ bằng *ngôn ngữ của riêng mình* và khiến máy tính thực hiện một việc — đó giống như một dạng siêu năng lực ở cấp độ cao hơn. Đó là một cảm giác tuyệt vời, và đến giờ vẫn vậy.

Tôi nhận ra mình có thể thiết kế một ngôn ngữ trông và hoạt động theo bất kỳ cách nào mình muốn. Giống như cả đời tôi học ở một trường tư bắt buộc mặc đồng phục, rồi một ngày chuyển sang trường công nơi tôi có thể mặc bất cứ gì mình thích. Tôi không cần dùng dấu ngoặc nhọn cho block? Tôi có thể dùng thứ khác ngoài dấu bằng cho phép gán? Tôi có thể làm object mà không cần class? Đa kế thừa *và* multimethod? Một ngôn ngữ động nhưng overload tĩnh theo arity?

Tất nhiên, tôi tận dụng sự tự do đó và “chạy hết ga”. Tôi đưa ra những quyết định thiết kế ngôn ngữ kỳ lạ và tùy tiện nhất: dùng dấu nháy đơn cho generics, bỏ dấu phẩy giữa các đối số, cơ chế resolve overload có thể thất bại ở runtime. Tôi làm khác chỉ vì… muốn khác.

Đây là một trải nghiệm rất vui mà tôi khuyến khích bạn thử. Chúng ta cần nhiều ngôn ngữ lập trình kỳ lạ, tiên phong hơn. Tôi muốn thấy nhiều “ngôn ngữ nghệ thuật” hơn. Thỉnh thoảng tôi vẫn tạo ra những ngôn ngữ “đồ chơi” kỳ quặc chỉ để vui.

*Tuy nhiên*, nếu mục tiêu của bạn là thành công — mà “thành công” được định nghĩa là có thật nhiều người dùng — thì ưu tiên của bạn sẽ khác. Khi đó, mục tiêu chính là đưa ngôn ngữ của bạn “nạp” vào não của càng nhiều người càng tốt. Điều này *rất khó*. Cần rất nhiều công sức để chuyển cú pháp và ngữ nghĩa của một ngôn ngữ từ máy tính vào hàng nghìn tỷ neuron.

Lập trình viên vốn dĩ thận trọng với thời gian của mình và cân nhắc kỹ xem ngôn ngữ nào đáng để “tải” vào bộ não sinh học. Họ không muốn phí thời gian cho một ngôn ngữ cuối cùng lại chẳng hữu ích. Vậy nên, với tư cách là nhà thiết kế ngôn ngữ, mục tiêu của bạn là mang đến cho họ càng nhiều sức mạnh ngôn ngữ càng tốt, với lượng kiến thức mới cần học là ít nhất.

Một cách tiếp cận tự nhiên là *đơn giản hóa*. Càng ít khái niệm và tính năng, khối lượng kiến thức cần học càng nhỏ. Đây là một trong những lý do các ngôn ngữ <span name="dynamic">script</span> tối giản thường thành công, dù không mạnh mẽ bằng các ngôn ngữ công nghiệp lớn — chúng dễ bắt đầu hơn, và khi đã “nằm trong não” ai đó, họ sẽ muốn tiếp tục dùng.

<aside name="dynamic">

Đặc biệt, đây là lợi thế lớn của ngôn ngữ kiểu động. Một ngôn ngữ kiểu tĩnh yêu cầu bạn học *hai* ngôn ngữ — ngữ nghĩa runtime và hệ thống kiểu tĩnh — trước khi bạn có thể bắt đầu khiến máy tính làm việc. Ngôn ngữ động chỉ yêu cầu bạn học phần đầu tiên.

Cuối cùng, khi chương trình đủ lớn, giá trị của phân tích tĩnh sẽ bù đắp công sức học “ngôn ngữ tĩnh” thứ hai đó, nhưng ban đầu thì lợi ích này không rõ ràng.

</aside>

Vấn đề của sự đơn giản là việc cắt bỏ tính năng thường đồng nghĩa hy sinh sức mạnh và khả năng biểu đạt. Có một nghệ thuật trong việc tìm ra những tính năng “đáng đồng tiền bát gạo”, nhưng nhiều ngôn ngữ tối giản đơn giản là làm được ít hơn.

Có một con đường khác để tránh phần lớn vấn đề này. Mấu chốt là nhận ra rằng người dùng không cần “nạp” toàn bộ ngôn ngữ của bạn vào đầu, *chỉ phần họ chưa biết*. Như tôi đã nói trong [một ghi chú thiết kế trước](parsing-expressions.html#design-note), học là quá trình truyền tải *phần chênh lệch* giữa những gì họ đã biết và những gì họ cần biết.

Nhiều người dùng tiềm năng của ngôn ngữ bạn đã biết một ngôn ngữ lập trình khác. Bất kỳ tính năng nào ngôn ngữ của bạn chia sẻ với ngôn ngữ đó gần như là “miễn phí” về mặt học tập. Nó đã ở trong đầu họ, họ chỉ cần nhận ra rằng ngôn ngữ của bạn làm điều tương tự.

Nói cách khác, *sự quen thuộc* là một công cụ quan trọng khác để giảm chi phí tiếp nhận ngôn ngữ. Tất nhiên, nếu bạn tối đa hóa yếu tố này, kết quả cuối cùng sẽ là một ngôn ngữ hoàn toàn giống hệt ngôn ngữ khác. Đó không phải là công thức thành công, vì khi đó chẳng có lý do gì để người dùng chuyển sang ngôn ngữ của bạn.

Vì vậy, bạn vẫn cần mang đến những điểm khác biệt hấp dẫn — những điều ngôn ngữ của bạn có thể làm mà ngôn ngữ khác không thể, hoặc ít nhất là không làm tốt bằng. Tôi tin đây là một trong những bài toán cân bằng cơ bản của thiết kế ngôn ngữ: giống các ngôn ngữ khác giúp giảm chi phí học, trong khi khác biệt lại tăng sức hút.

Tôi nghĩ về bài toán cân bằng này như một <span name="idiosyncracy">**ngân sách độ mới lạ**</span>, hay như Steve Klabnik gọi là “[ngân sách độ kỳ lạ](https://words.steveklabnik.com/the-language-strangeness-budget)”. Người dùng có một ngưỡng thấp cho tổng lượng “điều mới” mà họ sẵn sàng chấp nhận để học một ngôn ngữ mới. Vượt quá ngưỡng đó, họ sẽ bỏ đi.

<aside name="idiosyncracy">

Một khái niệm liên quan trong tâm lý học là [**idiosyncrasy credit**](https://en.wikipedia.org/wiki/Idiosyncrasy_credit) — ý tưởng rằng những người khác trong xã hội cho bạn một lượng giới hạn các “lệch chuẩn” so với quy tắc chung. Bạn “kiếm” được tín dụng bằng cách hòa nhập và làm những việc “thuộc nhóm”, rồi có thể “tiêu” nó cho những hành vi khác thường mà bình thường sẽ bị để ý. Nói cách khác, chứng minh rằng bạn “thuộc về nhóm” sẽ cho bạn quyền “giương cờ lập dị”, nhưng chỉ ở mức nào đó thôi.

</aside>

Bất cứ khi nào bạn thêm một thứ mới vào ngôn ngữ mà các ngôn ngữ khác không có, hoặc làm một việc theo cách khác so với các ngôn ngữ khác, bạn đang tiêu một phần ngân sách đó. Điều này là bình thường — bạn *cần* tiêu nó để khiến ngôn ngữ của mình hấp dẫn. Nhưng mục tiêu là tiêu *khôn ngoan*. Với mỗi tính năng hoặc khác biệt, hãy tự hỏi nó mang lại bao nhiêu sức hút cho ngôn ngữ, rồi đánh giá kỹ xem nó có “đáng tiền” không. Liệu thay đổi này có đủ giá trị để xứng đáng tiêu một phần ngân sách độ mới lạ?

Trên thực tế, tôi thấy điều này thường dẫn đến việc bạn khá bảo thủ với cú pháp nhưng mạo hiểm hơn với ngữ nghĩa. Dù việc “thay áo mới” rất vui, nhưng đổi dấu ngoặc nhọn sang một ký hiệu block khác hiếm khi mang lại nhiều sức mạnh thực sự cho ngôn ngữ, trong khi vẫn tiêu tốn ngân sách mới lạ. Khó để khác biệt về cú pháp “tự trả chi phí” cho mình.

Ngược lại, những ngữ nghĩa mới có thể tăng đáng kể sức mạnh của ngôn ngữ. Multimethod, mixin, trait, reflection, dependent type, metaprogramming ở runtime… có thể nâng tầm đáng kể những gì người dùng làm được với ngôn ngữ.

Tiếc là, cách tiếp cận bảo thủ này không vui bằng việc “thay đổi tất cả”. Nhưng bạn phải tự quyết định xem mình có muốn theo đuổi thành công đại chúng hay không. Không phải ai cũng cần trở thành một ban nhạc pop “thân thiện với radio”. Nếu bạn muốn ngôn ngữ của mình giống như free jazz hay drone metal và hài lòng với một lượng khán giả nhỏ hơn (nhưng có lẽ trung thành hơn), thì cứ làm thôi.