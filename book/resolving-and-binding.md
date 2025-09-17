> Thỉnh thoảng bạn thấy mình rơi vào một tình huống kỳ lạ. Bạn bước vào đó từng chút một theo cách tự nhiên nhất, nhưng khi đang ở giữa, bạn bỗng ngạc nhiên và tự hỏi làm sao mọi chuyện lại thành ra như vậy.
>
> <cite>Thor Heyerdahl, <em>Kon-Tiki</em></cite>

Ôi không! Hiện thực ngôn ngữ của chúng ta đang… thủng đáy! Hồi xa xưa khi [thêm biến và khối lệnh](statements-and-state.html), ta đã có cơ chế scoping gọn gàng, chặt chẽ. Nhưng khi [sau đó thêm closures](functions.html), một lỗ hổng đã xuất hiện trong interpreter vốn “chống nước” của ta. Phần lớn các chương trình thực tế khó mà lọt qua lỗ hổng này, nhưng với tư cách là những người hiện thực ngôn ngữ, ta thề sẽ quan tâm đến tính đúng đắn ngay cả ở những góc sâu và ẩm ướt nhất của ngữ nghĩa.

Trong cả chương này, ta sẽ khám phá lỗ hổng đó và cẩn thận bịt nó lại. Trong quá trình này, ta sẽ hiểu rõ hơn về *lexical scoping* như Lox và các ngôn ngữ thuộc họ C sử dụng. Ta cũng sẽ có dịp tìm hiểu về *semantic analysis* — một kỹ thuật mạnh mẽ để trích xuất ý nghĩa từ mã nguồn của người dùng mà không cần chạy nó.

## Static Scope

Ôn lại nhanh: Lox, giống hầu hết các ngôn ngữ hiện đại, dùng *lexical* scoping. Điều này có nghĩa là bạn có thể xác định một tên biến tham chiếu đến khai báo nào chỉ bằng cách đọc văn bản chương trình. Ví dụ:

```lox
var a = "outer";
{
  var a = "inner";
  print a;
}
```

Ở đây, ta biết `a` được in ra là biến được khai báo ở dòng ngay trước đó, chứ không phải biến toàn cục. Việc chạy chương trình không — *không thể* — ảnh hưởng đến điều này. Các quy tắc scope là một phần của ngữ nghĩa *tĩnh* của ngôn ngữ, vì vậy chúng còn được gọi là *static scope*.

Tôi chưa từng viết rõ các quy tắc scope đó, nhưng giờ là lúc cần <span name="precise">chính xác</span>:

<aside name="precise">

Điều này vẫn chưa đâu vào đâu so với một đặc tả ngôn ngữ thực sự. Những tài liệu đó phải rõ ràng đến mức ngay cả một người Sao Hỏa hay một lập trình viên cố tình phá hoại cũng buộc phải hiện thực đúng ngữ nghĩa nếu họ tuân thủ từng chữ trong đặc tả.

Sự chính xác này rất quan trọng khi một ngôn ngữ có thể được hiện thực bởi các công ty cạnh tranh, những bên muốn sản phẩm của mình không tương thích với sản phẩm khác để “giam” khách hàng trên nền tảng của họ. May mắn là trong cuốn sách này, ta có thể bỏ qua những trò mờ ám kiểu đó.

</aside>

**Một lần sử dụng biến sẽ tham chiếu đến khai báo trước đó có cùng tên trong scope trong cùng bao quanh biểu thức nơi biến được sử dụng.**

Có khá nhiều thứ để bóc tách ở đây:

*   Tôi nói “lần sử dụng biến” thay vì “biểu thức biến” để bao quát cả biểu thức biến và phép gán. Tương tự với “biểu thức nơi biến được sử dụng”.

*   “Trước đó” nghĩa là xuất hiện trước *trong văn bản chương trình*.

    ```lox
    var a = "outer";
    {
      print a;
      var a = "inner";
    }
    ```

    Ở đây, `a` được in ra là biến bên ngoài vì nó xuất hiện <span name="hoisting">trước</span> câu lệnh `print` sử dụng nó. Trong hầu hết các trường hợp, trong code tuyến tính, khai báo xuất hiện trước *trong văn bản* cũng sẽ xuất hiện trước *trong thời gian chạy*. Nhưng không phải lúc nào cũng vậy. Như ta sẽ thấy, hàm có thể trì hoãn một đoạn code sao cho thứ tự *thời gian động* khi execute không còn phản chiếu thứ tự *văn bản tĩnh* nữa.

    <aside name="hoisting">

    Trong JavaScript, các biến khai báo bằng `var` được “hoisting” ngầm lên đầu khối. Bất kỳ lần sử dụng tên đó trong khối sẽ tham chiếu đến biến đó, ngay cả khi lần sử dụng xuất hiện trước khai báo. Khi bạn viết thế này trong JavaScript:

    ```js
    {
      console.log(a);
      var a = "value";
    }
    ```

    Nó sẽ hoạt động như:

    ```js
    {
      var a; // Hoist.
      console.log(a);
      a = "value";
    }
    ```

    Điều này có nghĩa là trong một số trường hợp, bạn có thể đọc biến trước khi nó được khởi tạo — một nguồn gây lỗi khó chịu. Cú pháp `let` để khai báo biến được thêm vào sau này nhằm giải quyết vấn đề này.

    </aside>

*   “Trong cùng” ở đây là vì hiện tượng shadowing quen thuộc. Có thể có nhiều biến cùng tên trong các scope bao quanh, như:

    ```lox
    var a = "outer";
    {
      var a = "inner";
      print a;
    }
    ```

    Quy tắc của ta phân định trường hợp này bằng cách nói rằng scope trong cùng sẽ thắng.

Vì quy tắc này không đề cập đến bất kỳ hành vi runtime nào, nó ngụ ý rằng một biểu thức biến luôn tham chiếu đến cùng một khai báo trong suốt quá trình execute chương trình. Interpreter của ta cho đến giờ *hầu như* hiện thực đúng quy tắc này. Nhưng khi ta thêm closures, một lỗi đã len vào.

```lox
var a = "global";
{
  fun showA() {
    print a;
  }

  showA();
  var a = "block";
  showA();
}
```

<span name="tricky">Trước khi</span> bạn gõ và chạy đoạn này, hãy thử đoán xem nó *nên* in ra gì.

<aside name="tricky">

Tôi biết, đây là một chương trình hoàn toàn *dị*. Nó thật là *kỳ quặc*. Không ai tỉnh táo lại viết code như thế này. Nhưng đáng tiếc, nếu bạn gắn bó lâu với lĩnh vực ngôn ngữ lập trình, bạn sẽ dành nhiều thời gian hơn mình tưởng để xử lý những đoạn code dị” kiểu này.

</aside>

OK… hiểu ý chứ? Nếu bạn quen với closures trong các ngôn ngữ khác, hẳn bạn sẽ mong nó in ra `"global"` hai lần. Lần gọi đầu tiên tới `showA()` chắc chắn phải in `"global"` vì ta thậm chí chưa tới chỗ khai báo biến `a` bên trong. Và theo quy tắc rằng một biểu thức biến luôn được giải quyết tới cùng một biến, điều đó ngụ ý rằng lần gọi thứ hai tới `showA()` cũng phải in ra y như vậy.

Tiếc thay, nó lại in:

```text
global
block
```

Tôi muốn nhấn mạnh rằng chương trình này không hề gán lại bất kỳ biến nào và chỉ chứa đúng một câu lệnh `print`. Ấy vậy mà, câu lệnh `print` đó — cho một biến chưa từng được gán — lại in ra hai giá trị khác nhau ở hai thời điểm khác nhau. Rõ ràng là ta đã làm hỏng thứ gì đó.

### Scope & môi trường có thể thay đổi

Trong interpreter của ta, environment là biểu hiện động của static scope. Hai thứ này hầu như luôn đồng bộ với nhau — ta tạo một environment mới khi vào một scope mới, và bỏ nó đi khi rời scope. Có một thao tác khác ta thực hiện trên environment: ràng buộc (bind) một biến trong đó. Và đây chính là chỗ lỗi xuất hiện.

Hãy cùng đi qua ví dụ gây rắc rối này và xem environment trông thế nào ở từng bước. Đầu tiên, ta khai báo `a` trong global scope.

<img src="image/resolving-and-binding/environment-1.png" alt="The global environment with 'a' defined in it." />

Lúc này ta có một environment duy nhất với một biến duy nhất. Sau đó ta vào block và execute khai báo `showA()`.

<img src="image/resolving-and-binding/environment-2.png" alt="A block environment linking to the global one." />

Ta có một environment mới cho block. Trong đó, ta khai báo một tên `showA`, được bind tới đối tượng `LoxFunction` mà ta tạo ra để biểu diễn hàm. Đối tượng này có một trường `closure` lưu lại environment nơi hàm được khai báo, nên nó tham chiếu ngược về environment của block.

Giờ ta gọi `showA()`.

<img src="image/resolving-and-binding/environment-3.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the global environment." />

Interpreter tạo động một environment mới cho phần thân hàm `showA()`. Nó trống vì hàm này không khai báo biến nào. Cha của environment này là closure của hàm — environment của block bên ngoài.

Bên trong thân `showA()`, ta in giá trị của `a`. Interpreter tìm giá trị này bằng cách đi ngược chuỗi environment. Nó đi tới tận global environment mới tìm thấy và in `"global"`. Tuyệt.

Tiếp theo, ta khai báo biến `a` thứ hai, lần này bên trong block.

<img src="image/resolving-and-binding/environment-4.png" alt="The block environment has both 'a' and 'showA' now." />

Nó nằm trong cùng block — cùng scope — với `showA()`, nên nó được thêm vào cùng environment, cũng chính là environment mà closure của `showA()` tham chiếu tới. Đây là lúc mọi thứ trở nên thú vị. Ta gọi lại `showA()`.

<img src="image/resolving-and-binding/environment-5.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the block environment." />

Ta lại tạo một environment trống cho phần thân `showA()`, nối nó với closure đó, và chạy thân hàm. Khi interpreter đi ngược chuỗi environment để tìm `a`, giờ nó tìm thấy biến `a` *mới* trong environment của block. Thật tệ.

Tôi đã chọn hiện thực environment theo cách mà tôi hy vọng sẽ phù hợp với trực giác của bạn về scope. Ta thường nghĩ toàn bộ code trong một block nằm trong cùng một scope, nên interpreter dùng một environment duy nhất để biểu diễn nó. Mỗi environment là một hash table có thể thay đổi. Khi một biến cục bộ mới được khai báo, nó được thêm vào environment hiện có của scope đó.

Trực giác này, giống như nhiều thứ khác trong đời, không hoàn toàn đúng. Một block không nhất thiết là cùng một scope từ đầu đến cuối. Xem ví dụ:

```lox
{
  var a;
  // 1.
  var b;
  // 2.
}
```

Tại dòng đánh dấu 1, chỉ có `a` là trong scope. Tại dòng 2, cả `a` và `b` đều trong scope. Nếu bạn định nghĩa “scope” là tập hợp các khai báo, thì rõ ràng đây không phải cùng một scope — chúng không chứa cùng một tập khai báo. Giống như mỗi câu lệnh `var` <span name="split">tách</span> block thành hai scope riêng biệt: scope trước khi biến được khai báo và scope sau đó, bao gồm cả biến mới.

<aside name="split">

Một số ngôn ngữ thể hiện rõ sự tách này. Trong Scheme và ML, khi bạn khai báo biến cục bộ bằng `let`, bạn cũng đồng thời xác định rõ đoạn code tiếp theo nơi biến mới có hiệu lực. Không có khái niệm ngầm định “phần còn lại của block”.

</aside>

Nhưng trong hiện thực của ta, environment lại hoạt động như thể toàn bộ block là một scope duy nhất, chỉ là scope đó thay đổi theo thời gian. Closures thì không thích điều này. Khi một hàm được khai báo, nó lưu lại tham chiếu tới environment hiện tại. Hàm *đáng lẽ* phải lưu một bản chụp tĩnh (snapshot) của environment *tại thời điểm hàm được khai báo*. Nhưng thay vào đó, trong code Java, nó giữ tham chiếu tới chính đối tượng environment có thể thay đổi. Khi một biến được khai báo sau đó trong scope mà environment này tương ứng, closure sẽ thấy biến mới, dù khai báo đó *không* xuất hiện trước hàm.

### Persistent environment

Có một phong cách lập trình sử dụng cái gọi là **persistent data structure**. Không giống các cấu trúc dữ liệu “mềm” mà bạn quen thuộc trong lập trình mệnh lệnh, một persistent data structure không bao giờ bị thay đổi trực tiếp. Thay vào đó, bất kỳ “thay đổi” nào lên cấu trúc hiện có sẽ tạo ra một đối tượng <span name="copy">mới tinh</span> chứa toàn bộ dữ liệu gốc và phần thay đổi mới. Bản gốc vẫn giữ nguyên.

<aside name="copy">

Nghe có vẻ như điều này sẽ tốn rất nhiều bộ nhớ và thời gian để sao chép cấu trúc cho mỗi thao tác. Nhưng trên thực tế, persistent data structure chia sẻ phần lớn dữ liệu giữa các “bản sao” khác nhau.

</aside>

Nếu ta áp dụng kỹ thuật này cho `Environment`, thì mỗi khi bạn khai báo một biến, nó sẽ trả về một environment *mới* chứa tất cả các biến đã khai báo trước đó cùng với tên mới. Việc khai báo biến sẽ thực hiện “tách” ngầm định: bạn có một environment trước khi biến được khai báo và một environment sau đó:

<img src="image/resolving-and-binding/split.png" alt="Separate environments before and after the variable is declared." />

Một closure giữ tham chiếu tới instance `Environment` đang tồn tại khi hàm được khai báo. Vì mọi khai báo sau đó trong block sẽ tạo ra các đối tượng `Environment` mới, closure sẽ không thấy các biến mới và lỗi của ta sẽ được sửa.

Đây là một cách hợp lý để giải quyết vấn đề, và cũng là cách kinh điển để hiện thực environment trong các interpreter Scheme. Ta có thể làm vậy cho Lox, nhưng điều đó sẽ đồng nghĩa với việc phải quay lại và thay đổi một đống code hiện có.

Tôi sẽ không bắt bạn đi qua con đường đó. Ta sẽ giữ nguyên cách biểu diễn environment như hiện tại. Thay vì làm cho dữ liệu có cấu trúc tĩnh hơn, ta sẽ “nướng” phần giải quyết tĩnh (static resolution) vào ngay trong thao tác *truy cập*.

## Phân tích ngữ nghĩa (Semantic Analysis)

Interpreter của chúng ta **resolve** một biến — tức là lần theo xem nó tham chiếu tới khai báo nào — *mỗi lần* biểu thức biến đó được đánh giá. Nếu biến đó nằm trong một vòng lặp chạy cả nghìn lần, thì nó sẽ bị resolve lại cả nghìn lần.

Ta biết static scope có nghĩa là một lần sử dụng biến luôn resolve tới cùng một khai báo, và điều này có thể xác định chỉ bằng cách nhìn vào văn bản chương trình. Vậy thì, tại sao ta lại làm việc này động (dynamic) mỗi lần? Làm vậy không chỉ mở ra lỗ hổng dẫn tới lỗi khó chịu của ta, mà còn chậm một cách không cần thiết.

Giải pháp tốt hơn là resolve mỗi lần sử dụng biến *một lần duy nhất*. Viết một đoạn code để duyệt qua chương trình của người dùng, tìm mọi biến được nhắc tới, và xác định mỗi biến đó tham chiếu tới khai báo nào. Quá trình này là một ví dụ của **semantic analysis**. Nếu parser chỉ cho biết chương trình có đúng ngữ pháp hay không (một dạng *syntactic analysis*), thì semantic analysis đi xa hơn và bắt đầu tìm hiểu các phần của chương trình *thực sự* có nghĩa gì. Trong trường hợp này, phân tích của ta sẽ resolve các variable binding. Ta sẽ biết không chỉ rằng một biểu thức *là* một biến, mà còn biết *đó là biến nào*.

Có nhiều cách để lưu trữ mối liên kết giữa một biến và khai báo của nó. Khi tới interpreter C cho Lox, ta sẽ có một cách lưu trữ và truy cập biến cục bộ *hiệu quả hơn nhiều*. Nhưng với jlox, tôi muốn giảm thiểu tối đa tác động phụ lên codebase hiện tại. Tôi không muốn vứt bỏ một đống code vốn vẫn ổn.

Thay vào đó, ta sẽ lưu thông tin resolution theo cách tận dụng tối đa class `Environment` hiện có. Hãy nhớ lại cách các lần truy cập `a` được interpreter xử lý trong ví dụ gây lỗi.

<img src="image/resolving-and-binding/environment-3.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the global environment." />

Trong lần đánh giá đầu tiên (đúng), ta phải đi qua ba environment trong chuỗi trước khi tìm thấy khai báo toàn cục của `a`. Sau đó, khi biến `a` bên trong được khai báo trong block scope, nó che khuất (shadow) biến toàn cục.

<img src="image/resolving-and-binding/environment-5.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the block environment." />

Lần lookup tiếp theo đi qua chuỗi, tìm thấy `a` trong environment *thứ hai* và dừng lại ở đó. Mỗi environment tương ứng với một lexical scope nơi biến được khai báo. Nếu ta có thể đảm bảo một lần lookup biến luôn đi qua *cùng* số bước trong chuỗi environment, thì ta sẽ đảm bảo nó luôn tìm thấy cùng một biến trong cùng một scope mỗi lần.

Để “resolve” một lần sử dụng biến, ta chỉ cần tính xem biến được khai báo sẽ nằm cách bao nhiêu “bước” trong chuỗi environment. Câu hỏi thú vị là *khi nào* thực hiện phép tính này — hay nói cách khác, trong phần nào của hiện thực interpreter ta sẽ nhét đoạn code này vào?

Vì ta đang tính một thuộc tính tĩnh dựa trên cấu trúc mã nguồn, câu trả lời hiển nhiên là trong parser. Đó là nơi truyền thống để làm việc này, và cũng là nơi ta sẽ đặt nó sau này trong clox. Nó cũng có thể hoạt động ở đây, nhưng tôi muốn có cớ để cho bạn thấy một kỹ thuật khác. Ta sẽ viết resolver như một bước (pass) riêng.

### Bước resolve biến

Sau khi parser tạo ra syntax tree, nhưng trước khi interpreter bắt đầu execute, ta sẽ duyệt qua cây một lần để resolve tất cả các biến mà nó chứa. Các bước bổ sung giữa parsing và execution là chuyện thường gặp. Nếu Lox có kiểu tĩnh, ta có thể chèn một bộ kiểm tra kiểu vào đó. Các tối ưu hóa cũng thường được hiện thực trong các bước riêng như thế này. Nói chung, bất kỳ công việc nào không phụ thuộc vào trạng thái chỉ có ở runtime đều có thể làm theo cách này.

Bước resolve biến của ta hoạt động giống như một mini-interpreter. Nó duyệt cây, thăm từng node, nhưng static analysis khác với dynamic execution ở chỗ:

*   **Không có side effect.** Khi static analysis thăm một câu lệnh `print`, nó sẽ không thực sự in gì cả. Các lời gọi tới hàm native hoặc các thao tác tác động ra thế giới bên ngoài sẽ bị bỏ qua và không gây hiệu ứng.

*   **Không có control flow.** Vòng lặp chỉ được thăm <span name="fix">một lần</span>. Cả hai nhánh trong câu lệnh `if` đều được thăm. Các toán tử logic không short-circuit.

<aside name="fix">

Việc resolve biến chạm vào mỗi node một lần, nên hiệu năng của nó là *O(n)* với *n* là số node trong syntax tree. Các phân tích phức tạp hơn có thể có độ phức tạp lớn hơn, nhưng hầu hết đều được thiết kế cẩn thận để tuyến tính hoặc gần tuyến tính. Sẽ thật xấu hổ nếu compiler của bạn chậm theo cấp số nhân khi chương trình của người dùng lớn dần.

</aside>

## Lớp Resolver

Giống như mọi thứ trong Java, bước resolve biến của ta được hiện thân trong một class.

^code resolver

Vì resolver cần thăm mọi node trong syntax tree, nó hiện thực abstraction visitor mà ta đã có sẵn. Chỉ một vài loại node là đáng chú ý khi resolve biến:

*   Câu lệnh block tạo ra một scope mới cho các câu lệnh bên trong nó.

*   Khai báo hàm tạo ra một scope mới cho phần thân và bind các tham số trong scope đó.

*   Khai báo biến thêm một biến mới vào scope hiện tại.

*   Biểu thức biến và biểu thức gán cần được resolve biến.

Các node còn lại không làm gì đặc biệt, nhưng ta vẫn cần hiện thực các phương thức `visit` cho chúng để duyệt vào các cây con. Dù một biểu thức `+` *tự nó* không có biến nào để resolve, nhưng một trong hai toán hạng của nó có thể có.

### Resolve các khối lệnh (blocks)

Ta bắt đầu với block vì chúng tạo ra các local scope — nơi mọi “phép màu” diễn ra.

^code visit-block-stmt

Đoạn này bắt đầu một scope mới, duyệt vào các câu lệnh bên trong block, rồi bỏ scope đó đi. Phần thú vị nằm trong các hàm helper. Ta bắt đầu với cái đơn giản trước.

^code resolve-statements

Hàm này duyệt qua một danh sách các câu lệnh và resolve từng câu lệnh. Nó lại gọi tiếp:

^code resolve-stmt

Nhân tiện, ta thêm một overload khác mà sau này sẽ cần để resolve một biểu thức.

^code resolve-expr

Các phương thức này tương tự như `evaluate()` và `execute()` trong Interpreter — chúng áp dụng Visitor pattern lên node của syntax tree được truyền vào.

Phần hành vi thú vị thực sự nằm ở scope. Một block scope mới được tạo như sau:

^code begin-scope

Lexical scope được lồng nhau cả trong interpreter lẫn resolver. Chúng hoạt động như một stack. Interpreter hiện thực stack này bằng một linked list — chuỗi các đối tượng `Environment`. Trong resolver, ta dùng hẳn một `Stack` của Java.

^code scopes-field (1 before, 2 after)

Trường này lưu stack các scope hiện đang… trong scope. Mỗi phần tử trong stack là một `Map` biểu diễn một block scope duy nhất. Key, giống như trong `Environment`, là tên biến. Value là Boolean, lý do sẽ được giải thích ngay sau đây.

Stack scope này chỉ dùng cho các local block scope. Các biến khai báo ở cấp cao nhất (global scope) không được resolver theo dõi vì chúng mang tính động hơn trong Lox. Khi resolve một biến, nếu không tìm thấy nó trong stack các local scope, ta giả định nó là biến global.

Vì scope được lưu trong một stack tường minh, việc thoát khỏi một scope khá đơn giản.

^code end-scope

Giờ ta có thể push và pop một stack các scope rỗng. Hãy đặt một vài thứ vào chúng.

### Resolve khai báo biến

Resolve một khai báo biến sẽ thêm một entry mới vào map của scope trong cùng hiện tại. Nghe có vẻ đơn giản, nhưng ta cần một chút “múa” ở đây.

^code visit-var-stmt

Ta tách việc bind thành hai bước: declare rồi define, để xử lý các trường hợp “oái oăm” như sau:

```lox
var a = "outer";
{
  var a = a;
}
```

Điều gì xảy ra khi phần initializer của một biến cục bộ lại tham chiếu tới một biến có cùng tên với biến đang được khai báo? Ta có vài lựa chọn:

1.  **Chạy initializer, rồi mới đưa biến mới vào scope.** Khi đó, biến cục bộ `a` mới sẽ được khởi tạo bằng `"outer"`, giá trị của biến `a` *global*. Nói cách khác, khai báo trước đó sẽ được “desugar” thành:

    ```lox
    var temp = a; // Chạy initializer.
    var a;        // Khai báo biến.
    a = temp;     // Khởi tạo.
    ```

2.  **Đưa biến mới vào scope, rồi chạy initializer.** Nghĩa là bạn có thể “thấy” biến trước khi nó được khởi tạo, nên ta cần xác định giá trị của nó lúc đó là gì. Có lẽ là `nil`. Khi đó, biến cục bộ `a` mới sẽ được gán lại chính giá trị khởi tạo ngầm định của nó, `nil`. Desugar sẽ trông như:

    ```lox
    var a; // Định nghĩa biến.
    a = a; // Chạy initializer.
    ```

3.  **Báo lỗi nếu tham chiếu tới biến trong chính initializer của nó.** Interpreter sẽ fail ở compile time hoặc runtime nếu initializer nhắc tới biến đang được khởi tạo.

Hai lựa chọn đầu có giống thứ mà người dùng *thực sự* muốn không? Shadowing là hiếm và thường là lỗi, nên việc khởi tạo một biến shadow dựa trên giá trị của biến bị shadow có vẻ khó là chủ ý.

Lựa chọn thứ hai còn ít hữu ích hơn. Biến mới sẽ *luôn* có giá trị `nil`. Không bao giờ có lý do để nhắc tới nó bằng tên. Bạn có thể dùng `nil` tường minh thay thế.

Vì hai lựa chọn đầu dễ che giấu lỗi của người dùng, ta sẽ chọn cách thứ ba. Hơn nữa, ta sẽ biến nó thành lỗi compile-time thay vì runtime. Như vậy, người dùng sẽ được cảnh báo trước khi code chạy.

Để làm được điều đó, khi duyệt các biểu thức, ta cần biết mình có đang ở trong initializer của một biến nào đó hay không. Ta làm điều này bằng cách tách việc bind thành hai bước. Bước đầu tiên là **declare** nó.

^code declare

Khai báo sẽ thêm biến vào scope trong cùng để nó shadow bất kỳ biến bên ngoài nào và để ta biết biến đó tồn tại. Ta đánh dấu nó là “chưa sẵn sàng” bằng cách bind tên của nó với `false` trong scope map. Giá trị gắn với một key trong scope map biểu thị việc ta đã hoàn tất resolve initializer của biến đó hay chưa.

Sau khi khai báo biến, ta resolve biểu thức initializer của nó trong cùng scope đó — nơi biến mới tồn tại nhưng chưa thể dùng. Khi biểu thức initializer xong, biến đã sẵn sàng để dùng. Ta làm điều này bằng cách **define** nó.

^code define

Ta đặt giá trị của biến trong scope map thành `true` để đánh dấu nó đã được khởi tạo đầy đủ và sẵn sàng sử dụng. Nó đã “sống” rồi!

### Resolve biểu thức biến

Khai báo biến — và cả khai báo hàm (mà ta sẽ nói tới sau) — sẽ ghi vào scope map. Các map này sẽ được đọc khi ta resolve các biểu thức biến.

^code visit-variable-expr

Đầu tiên, ta kiểm tra xem biến có đang được truy cập bên trong chính initializer của nó hay không. Đây là lúc giá trị trong scope map phát huy tác dụng. Nếu biến tồn tại trong scope hiện tại nhưng giá trị của nó là `false`, nghĩa là ta đã khai báo nhưng chưa define nó. Ta sẽ báo lỗi.

Sau khi kiểm tra xong, ta thực sự resolve biến đó bằng helper này:

^code resolve-local

Nhìn quen quen đúng không? Nó khá giống code trong `Environment` để evaluate một biến. Ta bắt đầu từ scope trong cùng và đi ra ngoài, tìm trong từng map xem có tên khớp không. Nếu tìm thấy biến, ta resolve nó, truyền vào số scope giữa scope trong cùng hiện tại và scope nơi biến được tìm thấy. Vậy nên, nếu biến được tìm thấy trong scope hiện tại, ta truyền vào 0. Nếu nó nằm trong scope bao ngoài ngay lập tức, là 1. Bạn hiểu ý rồi đấy.

Nếu ta duyệt qua tất cả block scope mà vẫn không tìm thấy biến, ta để nó chưa resolve và giả định nó là biến global. Việc hiện thực phương thức `resolve()` đó ta sẽ nói sau. Giờ thì tiếp tục xử lý các node cú pháp khác.

### Resolve biểu thức gán

Biểu thức khác tham chiếu tới biến là phép gán. Resolve nó như sau:

^code visit-assign-expr

Đầu tiên, ta resolve biểu thức giá trị được gán, phòng khi nó cũng chứa tham chiếu tới biến khác. Sau đó, ta dùng lại phương thức `resolveLocal()` để resolve biến đang được gán.

### Resolve khai báo hàm

Cuối cùng là hàm. Hàm vừa bind tên vừa tạo ra một scope. Tên của hàm được bind trong scope bao quanh nơi hàm được khai báo. Khi ta bước vào thân hàm, ta cũng bind các tham số của nó vào scope bên trong của hàm.

^code visit-function-stmt

Tương tự `visitVariableStmt()`, ta declare và define tên hàm trong scope hiện tại. Nhưng khác với biến, ta define tên ngay lập tức, trước khi resolve thân hàm. Điều này cho phép hàm tự gọi đệ quy bên trong thân của chính nó.

Sau đó, ta resolve thân hàm bằng cách này:

^code resolve-function

Đây là một phương thức riêng vì sau này ta cũng sẽ dùng nó để resolve các method của Lox khi thêm class. Nó tạo một scope mới cho thân hàm rồi bind biến cho từng tham số của hàm.

Khi xong, nó resolve thân hàm trong scope đó. Điều này khác với cách interpreter xử lý khai báo hàm. Ở *runtime*, khai báo hàm không làm gì với thân hàm cả. Thân hàm chỉ được “đụng” tới khi hàm được gọi. Trong *static analysis*, ta duyệt ngay vào thân hàm tại chỗ.

### Resolve các node cú pháp khác

Vậy là ta đã xử lý những góc thú vị của ngữ pháp. Ta xử lý mọi nơi biến được khai báo, đọc hoặc ghi, và mọi nơi scope được tạo hoặc bị hủy. Dù không bị ảnh hưởng bởi việc resolve biến, ta vẫn cần các phương thức `visit` cho tất cả node cú pháp khác để đệ quy vào các cây con của chúng. <span name="boring">Xin lỗi</span> đoạn này hơi chán, nhưng hãy kiên nhẫn. Ta sẽ đi kiểu “top-down” và bắt đầu với statement.

<aside name="boring">

Tôi đã nói cuốn sách này sẽ có từng dòng code của các interpreter. Tôi đâu có nói tất cả đều thú vị.

</aside>

Một expression statement chứa một biểu thức duy nhất để duyệt.

^code visit-expression-stmt

Một if statement có một biểu thức điều kiện và một hoặc hai statement cho các nhánh.

^code visit-if-stmt

Ở đây, ta thấy sự khác biệt giữa resolution và interpretation. Khi resolve một `if`, không có control flow. Ta resolve điều kiện và *cả hai* nhánh. Trong khi execute động chỉ bước vào nhánh *được* chạy, static analysis thì thận trọng — nó phân tích mọi nhánh *có thể* được chạy. Vì ở runtime có thể đi vào bất kỳ nhánh nào, ta resolve cả hai.

Giống như expression statement, một `print` statement chứa một subexpression duy nhất.

^code visit-print-stmt

`return` cũng tương tự.

^code visit-return-stmt

Như với `if`, với `while` ta resolve điều kiện và resolve thân vòng lặp đúng một lần.

^code visit-while-stmt

Vậy là xong các statement. Giờ sang expression...

Người bạn cũ binary expression. Ta duyệt và resolve cả hai toán hạng.

^code visit-binary-expr

Call cũng tương tự — ta duyệt danh sách đối số và resolve tất cả. Thứ được gọi cũng là một expression (thường là variable expression), nên cũng được resolve.

^code visit-call-expr

Ngoặc đơn thì dễ.

^code visit-grouping-expr

Literal thì dễ nhất.

^code visit-literal-expr

Literal expression không nhắc tới biến nào và không chứa subexpression, nên không có gì để làm.

Vì static analysis không có control flow hay short-circuiting, logical expression giống hệt các toán tử nhị phân khác.

^code visit-logical-expr

Và cuối cùng, node cuối. Ta resolve toán hạng duy nhất của nó.

^code visit-unary-expr

Với tất cả các phương thức `visit` này, Java compiler sẽ hài lòng rằng `Resolver` đã hiện thực đầy đủ `Stmt.Visitor` và `Expr.Visitor`. Đây là lúc thích hợp để nghỉ ngơi, ăn nhẹ, hoặc chợp mắt một chút.

## Execute các biến đã được resolve

Hãy xem resolver của ta hữu ích thế nào. Mỗi lần nó thăm một biến, nó sẽ báo cho interpreter biết có bao nhiêu scope giữa scope hiện tại và scope nơi biến đó được khai báo. Khi runtime, con số này tương ứng chính xác với số lượng *environment* giữa environment hiện tại và environment bao ngoài nơi interpreter có thể tìm thấy giá trị của biến. Resolver chuyển con số đó cho interpreter bằng cách gọi:

^code resolve

Ta muốn lưu thông tin resolution này ở đâu đó để có thể dùng khi biểu thức biến hoặc gán được execute sau này, nhưng ở đâu? Một chỗ hiển nhiên là ngay trong node của syntax tree. Đây là một cách ổn, và cũng là nơi nhiều compiler lưu kết quả của các phân tích kiểu này.

Ta có thể làm vậy, nhưng sẽ phải đụng chạm vào trình sinh syntax tree của mình. Thay vào đó, ta sẽ dùng một cách phổ biến khác: lưu nó ở <span name="side">bên ngoài</span> trong một map liên kết mỗi node của syntax tree với dữ liệu đã được resolve của nó.

<aside name="side">

Tôi *nghĩ* mình đã nghe ai đó gọi map này là “side table” vì nó là một cấu trúc dữ liệu dạng bảng lưu trữ dữ liệu tách biệt với các đối tượng mà nó liên quan. Nhưng mỗi lần tôi thử Google cụm từ đó, tôi lại nhận được toàn trang về… đồ nội thất.

</aside>

Các công cụ tương tác như IDE thường parse lại và resolve lại từng phần của chương trình người dùng một cách gia tăng. Sẽ khó tìm ra tất cả các phần trạng thái cần tính toán lại khi chúng ẩn sâu trong “tán lá” của syntax tree. Một lợi ích của việc lưu dữ liệu này bên ngoài các node là ta có thể dễ dàng *loại bỏ* nó — chỉ cần xóa map.

^code locals-field (1 before, 2 after)

Bạn có thể nghĩ rằng ta cần một cấu trúc cây lồng nhau nào đó để tránh nhầm lẫn khi có nhiều biểu thức cùng tham chiếu tới một biến, nhưng mỗi node biểu thức là một đối tượng Java riêng với định danh duy nhất của nó. Một map đơn khối duy nhất sẽ không gặp vấn đề gì trong việc giữ chúng tách biệt.

Như thường lệ, dùng một collection yêu cầu ta import một vài tên.

^code import-hash-map (1 before, 1 after)

Và:

^code import-map (1 before, 2 after)

### Truy cập một biến đã được resolve

Interpreter của ta giờ đã có quyền truy cập vị trí đã resolve của từng biến. Cuối cùng thì ta cũng được dùng nó. Ta thay thế phương thức `visit` cho biểu thức biến bằng:

^code call-look-up-variable (1 before, 1 after)

Phương thức này ủy quyền cho:

^code look-up-variable

Có vài thứ diễn ra ở đây. Đầu tiên, ta tra khoảng cách đã resolve trong map. Nhớ rằng ta chỉ resolve các biến *local*. Biến global được xử lý đặc biệt và không nằm trong map (vì thế map mới tên là `locals`). Vậy nên, nếu không tìm thấy khoảng cách trong map, chắc chắn nó là biến global. Trong trường hợp đó, ta tra nó một cách động, trực tiếp trong global environment. Nếu biến chưa được định nghĩa, điều đó sẽ ném ra lỗi runtime.

Nếu ta *có* khoảng cách, tức là biến local, và ta có thể tận dụng kết quả của phân tích tĩnh. Thay vì gọi `get()`, ta gọi phương thức mới này trên `Environment`:

^code get-at

Phương thức `get()` cũ sẽ duyệt động qua chuỗi các environment bao ngoài, rà soát từng cái để xem biến có “ẩn” ở đâu đó không. Nhưng giờ ta biết chính xác environment nào trong chuỗi sẽ chứa biến. Ta tiếp cận nó bằng helper này:

^code ancestor

Helper này đi một số bước cố định lên chuỗi cha và trả về environment ở đó. Khi đã có environment đó, `getAt()` chỉ đơn giản trả về giá trị của biến trong map của environment đó. Nó thậm chí không cần kiểm tra xem biến có ở đó không — ta biết chắc là có vì resolver đã tìm thấy nó từ trước.

<aside name="coupled">

Cách interpreter giả định biến có trong map này giống như đang “bay mù”. Code interpreter tin tưởng rằng resolver đã làm đúng việc và resolve biến chính xác. Điều này ngụ ý một sự phụ thuộc chặt chẽ giữa hai class này. Trong resolver, mỗi dòng code đụng tới scope phải có một dòng tương ứng trong interpreter để chỉnh sửa environment.

Tôi đã cảm nhận sự phụ thuộc này trực tiếp vì khi viết code cho cuốn sách, tôi đã gặp vài lỗi tinh vi khi resolver và interpreter hơi lệch nhau. Lần ra chúng khá khó khăn. Một công cụ giúp việc này dễ hơn là để interpreter kiểm tra rõ ràng — dùng câu lệnh `assert` của Java hoặc công cụ xác thực khác — hợp đồng mà nó mong resolver đã đảm bảo.

</aside>

### Gán giá trị cho một biến đã được resolve

Ta cũng có thể sử dụng một biến bằng cách gán giá trị cho nó. Việc thay đổi khi thăm một biểu thức gán cũng tương tự.

^code resolved-assign (2 before, 1 after)

Một lần nữa, ta tra khoảng cách scope của biến. Nếu không tìm thấy, ta giả định nó là biến global và xử lý giống như trước. Ngược lại, ta gọi phương thức mới này:

^code assign-at

Nếu `getAt()` là phiên bản của `get()`, thì `assignAt()` là phiên bản của `assign()`. Nó đi qua một số lượng cố định các environment, rồi nhét giá trị mới vào map của environment đó.

Đó là tất cả thay đổi đối với Interpreter. Đây là lý do tôi chọn cách biểu diễn dữ liệu đã resolve sao cho ít xâm lấn nhất. Tất cả các node còn lại vẫn hoạt động như trước. Ngay cả code để chỉnh sửa environment cũng không thay đổi.

### Chạy resolver

Tất nhiên, ta cần thực sự *chạy* resolver. Ta chèn bước mới này sau khi parser đã hoàn thành công việc của nó.

^code create-resolver (3 before, 1 after)

Ta không chạy resolver nếu có bất kỳ lỗi parse nào. Nếu code có lỗi cú pháp, nó sẽ không bao giờ chạy, nên việc resolve là vô nghĩa. Nếu cú pháp sạch, ta bảo resolver làm việc của nó. Resolver có tham chiếu tới interpreter và chèn trực tiếp dữ liệu resolution vào đó khi nó duyệt qua các biến. Khi interpreter chạy tiếp theo, nó đã có mọi thứ cần thiết.

Ít nhất, điều đó đúng nếu resolver *thành công*. Nhưng nếu có lỗi trong quá trình resolve thì sao?

## Lỗi khi resolve

Vì ta đang thực hiện một bước phân tích ngữ nghĩa, ta có cơ hội làm cho ngữ nghĩa của Lox chính xác hơn, và giúp người dùng bắt lỗi sớm trước khi chạy code. Xem ví dụ “xấu” này:

```lox
fun bad() {
  var a = "first";
  var a = "second";
}
```

Ta cho phép khai báo nhiều biến cùng tên trong scope *global*, nhưng làm vậy trong scope cục bộ thì có lẽ là một sai lầm. Nếu họ biết biến đã tồn tại, họ sẽ gán cho nó thay vì dùng `var`. Và nếu họ *không* biết nó tồn tại, họ có lẽ cũng không định ghi đè biến trước đó.

Ta có thể phát hiện lỗi này một cách tĩnh khi resolve.

^code duplicate-variable (1 before, 1 after)

Khi ta khai báo một biến trong scope cục bộ, ta đã biết tên của mọi biến được khai báo trước đó trong cùng scope. Nếu thấy trùng, ta báo lỗi.

### Lỗi return không hợp lệ

Đây là một đoạn script “khó chịu” khác:

```lox
return "at top level";
```

Nó execute một câu lệnh `return`, nhưng thậm chí không nằm trong hàm nào cả. Đây là code ở cấp cao nhất. Tôi không biết người dùng *nghĩ* điều gì sẽ xảy ra, nhưng tôi không nghĩ ta muốn Lox cho phép điều này.

Ta có thể mở rộng resolver để phát hiện điều này một cách tĩnh. Giống như cách ta theo dõi scope khi duyệt cây, ta có thể theo dõi xem code hiện tại ta đang thăm có nằm trong một khai báo hàm hay không.

^code function-type-field (1 before, 2 after)

Thay vì một Boolean đơn giản, ta dùng một enum “vui vẻ” này:

^code function-type

Bây giờ thì có vẻ hơi thừa, nhưng sau này ta sẽ thêm vài trường hợp nữa và nó sẽ hợp lý hơn. Khi ta resolve một khai báo hàm, ta truyền giá trị này vào.

^code pass-function-type (2 before, 1 after)

Trong `resolveFunction()`, ta nhận tham số đó và lưu vào field trước khi resolve thân hàm.

^code set-current-function (1 after)

Trước tiên, ta lưu giá trị cũ của field vào một biến cục bộ. Nhớ rằng, Lox có hàm cục bộ, nên bạn có thể lồng khai báo hàm sâu tùy ý. Ta cần theo dõi không chỉ là đang ở trong một hàm, mà còn *bao nhiêu* hàm.

Ta có thể dùng một stack tường minh của các giá trị FunctionType cho việc này, nhưng thay vào đó ta tận dụng JVM. Ta lưu giá trị cũ vào một biến cục bộ trên Java stack. Khi resolve xong thân hàm, ta khôi phục field về giá trị đó.

^code restore-current-function (1 before, 1 after)

Giờ thì ta luôn biết mình có đang ở trong một khai báo hàm hay không, và ta kiểm tra điều đó khi resolve một câu lệnh `return`.

^code return-from-top (1 before, 1 after)

Hay ho, đúng không?

Còn một phần nữa. Quay lại class `Lox` chính, nơi kết nối mọi thứ lại với nhau, ta cẩn thận không chạy interpreter nếu gặp bất kỳ lỗi parse nào. Việc kiểm tra này chạy *trước* resolver để ta không cố resolve code sai cú pháp.

Nhưng ta cũng cần bỏ qua interpreter nếu có lỗi khi resolve, nên ta thêm *một* kiểm tra nữa.

^code resolution-error (1 before, 2 after)

Bạn có thể tưởng tượng thêm nhiều phân tích khác ở đây. Ví dụ, nếu ta thêm câu lệnh `break` vào Lox, ta có thể muốn đảm bảo chúng chỉ được dùng bên trong vòng lặp.

Ta có thể tiến xa hơn và báo cảnh báo cho code không hẳn là *sai* nhưng có lẽ không hữu ích. Ví dụ, nhiều IDE sẽ cảnh báo nếu bạn có code không thể tới được sau một câu lệnh `return`, hoặc một biến cục bộ mà giá trị của nó không bao giờ được đọc. Tất cả những điều đó khá dễ để thêm vào bước duyệt tĩnh này, hoặc như các bước <span name="separate">riêng biệt</span>.

<aside name="separate">

Việc quyết định gộp bao nhiêu loại phân tích khác nhau vào một pass duy nhất là một lựa chọn khó. Nhiều pass nhỏ, tách biệt, mỗi pass đảm nhận một trách nhiệm riêng, sẽ dễ hiện thực và bảo trì hơn. Tuy nhiên, việc duyệt qua syntax tree bản thân nó cũng có chi phí runtime đáng kể, nên gộp nhiều phân tích vào một pass thường sẽ nhanh hơn.

</aside>

Nhưng hiện tại, ta sẽ giữ nguyên mức phân tích hạn chế đó. Phần quan trọng là ta đã sửa được lỗi “góc cạnh” khó chịu kia, dù có thể bạn sẽ ngạc nhiên là phải tốn nhiều công sức đến vậy mới làm được.

<div class="challenges">

## Thử thách

1.  Tại sao có thể an toàn khi define ngay lập tức biến gắn với tên của một hàm, trong khi các biến khác phải đợi đến sau khi được khởi tạo mới có thể sử dụng?

2.  Các ngôn ngữ khác mà bạn biết xử lý thế nào với biến cục bộ tham chiếu tới chính tên đó trong initializer, như:

    ```lox
    var a = "outer";
    {
      var a = a;
    }
    ```

    Đây có phải là lỗi runtime? Lỗi compile? Được phép? Họ có xử lý biến global khác đi không? Bạn có đồng ý với lựa chọn của họ không? Hãy giải thích câu trả lời của bạn.

3.  Mở rộng resolver để báo lỗi nếu một biến cục bộ không bao giờ được sử dụng.

4.  Resolver của ta tính toán *biến* nằm trong environment nào, nhưng nó vẫn được tra cứu theo tên trong map. Một cách biểu diễn environment hiệu quả hơn là lưu biến cục bộ trong một mảng và tra cứu theo chỉ số.

    Mở rộng resolver để gán một chỉ số duy nhất cho mỗi biến cục bộ được khai báo trong một scope. Khi resolve một lần truy cập biến, tra cứu cả scope chứa biến và chỉ số của nó rồi lưu lại. Trong interpreter, dùng thông tin đó để truy cập nhanh biến theo chỉ số thay vì dùng map.

</div>