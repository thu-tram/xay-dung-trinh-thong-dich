
> Có gì tuyệt vời hơn việc bạn làm bữa sáng cho ai đó?
>
> <cite>Anthony Bourdain</cite>

Chúng ta sẽ dành phần còn lại của cuốn sách này để soi sáng mọi ngóc ngách, dù tối tăm hay kỳ lạ, của ngôn ngữ Lox. Nhưng sẽ thật tàn nhẫn nếu bắt bạn ngay lập tức cắm đầu viết code cho interpreter mà chưa cho bạn thấy trước một chút về thứ mà chúng ta sẽ tạo ra.

Đồng thời, tôi cũng không muốn lôi bạn qua hàng đống lý thuyết luật lệ ngôn ngữ và văn phong đặc tả khô khan trước khi bạn được chạm tay vào <span name="home">trình soạn thảo</span> của mình. Vì vậy, đây sẽ là một phần giới thiệu nhẹ nhàng, thân thiện về Lox. Nó sẽ bỏ qua nhiều chi tiết và các trường hợp đặc biệt. Chúng ta sẽ có plenty of time để bàn về chúng sau.

<aside name="home">

Một hướng dẫn sẽ chẳng vui chút nào nếu bạn không thể tự mình thử code. Tiếc là, bạn chưa có interpreter cho Lox, vì bạn chưa xây dựng nó!

Đừng lo. Bạn có thể dùng [của tôi](https://github.com/munificent/craftinginterpreters).

</aside>

## Hello, Lox

Đây là “miếng nếm” đầu tiên của bạn với <span name="salmon">Lox</span>:

<aside name="salmon">

Ý tôi là “nếm” Lox — ngôn ngữ lập trình ấy. Tôi không biết bạn đã từng ăn cá hồi xông khói lạnh chưa. Nếu chưa, bạn cũng nên thử món đó.

</aside>

```lox
// Your first Lox program!
print "Hello, world!";
```

Như dòng comment `//` và dấu chấm phẩy ở cuối gợi ý, cú pháp của Lox thuộc họ C. (Không có dấu ngoặc đơn quanh chuỗi vì `print` là một câu lệnh built-in, không phải hàm trong thư viện.)

Giờ, tôi sẽ không nói rằng <span name="c">C</span> có cú pháp *tuyệt vời*. Nếu muốn thứ gì đó thanh lịch, có lẽ ta sẽ bắt chước Pascal hoặc Smalltalk. Nếu muốn tối giản kiểu “nội thất Bắc Âu”, ta sẽ chọn Scheme. Mỗi cái đều có ưu điểm riêng.

<aside name="c">

Tôi chắc chắn là có phần thiên vị, nhưng tôi nghĩ cú pháp của Lox khá gọn gàng. Những vấn đề ngữ pháp tệ nhất của C thường xoay quanh kiểu dữ liệu. Dennis Ritchie từng có ý tưởng gọi là “[declaration reflects use](http://softwareengineering.stackexchange.com/questions/117024/why-was-the-c-syntax-for-arrays-pointers-and-functions-designed-this-way)”, nghĩa là khai báo biến phản ánh các thao tác bạn cần thực hiện trên biến đó để lấy ra giá trị kiểu cơ bản. Ý tưởng thông minh đấy, nhưng tôi không nghĩ nó hoạt động tốt trong thực tế.

Lox không có static types, nên chúng ta tránh được vấn đề này.

</aside>

Điều mà cú pháp giống C mang lại — và thường rất giá trị trong một ngôn ngữ — là *sự quen thuộc*. Tôi biết bạn đã quen với phong cách này, vì hai ngôn ngữ mà chúng ta sẽ dùng để *cài đặt* Lox — Java và C — cũng kế thừa nó. Dùng cú pháp tương tự cho Lox giúp bạn bớt đi một thứ phải học.

## A High-Level Language

Dù cuốn sách này đã dài hơn tôi mong muốn, nó vẫn không đủ chỗ để chứa một ngôn ngữ khổng lồ như Java. Để có thể đưa vào hai bản cài đặt hoàn chỉnh của Lox, bản thân Lox phải khá gọn nhẹ.

Khi nghĩ về những ngôn ngữ nhỏ nhưng hữu ích, tôi nhớ đến các ngôn ngữ “scripting” bậc cao như <span name="js">JavaScript</span>, Scheme và Lua. Trong ba cái đó, Lox trông giống JavaScript nhất, chủ yếu vì hầu hết các ngôn ngữ cú pháp C đều như vậy. Như chúng ta sẽ thấy sau, cách Lox xử lý scoping lại gần với Scheme. Phiên bản Lox viết bằng C mà chúng ta sẽ xây dựng trong [Part III](a-bytecode-virtual-machine.html) chịu ảnh hưởng lớn từ cách cài đặt gọn gàng, hiệu quả của Lua.

<aside name="js">

Giờ đây JavaScript đã “thống trị thế giới” và được dùng để xây dựng những ứng dụng khổng lồ, thật khó để nghĩ về nó như một “ngôn ngữ scripting nhỏ bé”. Nhưng Brendan Eich đã viết bản interpreter JS đầu tiên cho Netscape Navigator chỉ trong *mười ngày* để làm các nút bấm trên trang web biết… nhảy múa. JavaScript đã trưởng thành nhiều từ đó, nhưng từng có thời nó chỉ là một ngôn ngữ dễ thương, nhỏ gọn.

Vì Eich “chắp vá” JS từ nguyên liệu thô sơ và thời gian ít ỏi chẳng khác gì một tập MacGyver, nên nó có vài góc cạnh ngữ nghĩa kỳ quặc, nơi mà “băng keo” và “kẹp giấy” lộ ra. Ví dụ như variable hoisting, `this` được bind động, mảng có lỗ, và các phép chuyển đổi ngầm.

Tôi thì có thời gian thong thả hơn khi làm Lox, nên nó sẽ sạch sẽ hơn một chút.

</aside>

Lox còn chia sẻ hai đặc điểm khác với ba ngôn ngữ kia:

### Dynamic typing

Lox là ngôn ngữ dynamic typing. Biến có thể lưu giá trị thuộc bất kỳ kiểu nào, và một biến thậm chí có thể lưu các giá trị thuộc kiểu khác nhau ở những thời điểm khác nhau. Nếu bạn thử thực hiện một phép toán trên các giá trị sai kiểu — ví dụ, chia một số cho một chuỗi — thì lỗi sẽ được phát hiện và báo ngay tại runtime.

Có rất nhiều lý do để thích <span name="static">static</span> types, nhưng chúng không đủ để vượt qua những lý do thực dụng khi chọn dynamic types cho Lox. Hệ thống static type tốn rất nhiều công sức để học và cài đặt. Bỏ qua nó giúp ngôn ngữ đơn giản hơn và cuốn sách ngắn hơn. Chúng ta sẽ có interpreter chạy được code sớm hơn nếu dời việc kiểm tra kiểu sang runtime.

<aside name="static">

Suy cho cùng, hai ngôn ngữ mà chúng ta sẽ dùng để *cài đặt* Lox đều là statically typed.

</aside>


### Automatic memory management

Các ngôn ngữ bậc cao ra đời để loại bỏ những công việc tẻ nhạt, dễ gây lỗi ở mức thấp, và còn gì nhàm chán hơn việc phải tự tay quản lý cấp phát và giải phóng bộ nhớ? Chẳng ai thức dậy và chào đón ánh mặt trời buổi sáng với câu: “Hôm nay mình nóng lòng muốn tìm đúng chỗ để gọi `free()` cho từng byte bộ nhớ mình cấp phát quá!”

Có hai <span name="gc">kỹ thuật</span> chính để quản lý bộ nhớ: **reference counting** và **tracing garbage collection** (thường được gọi ngắn gọn là **garbage collection** hoặc **GC**). Reference counting đơn giản hơn nhiều để cài đặt — tôi nghĩ đó là lý do Perl, PHP và Python ban đầu đều dùng nó. Nhưng theo thời gian, những hạn chế của reference counting trở nên quá phiền toái. Tất cả các ngôn ngữ đó cuối cùng đều phải bổ sung một tracing GC đầy đủ, hoặc ít nhất là đủ để dọn dẹp các vòng tham chiếu giữa các object.

<aside name="gc">

Trên thực tế, reference counting và tracing giống như hai đầu của một phổ liên tục hơn là hai thái cực đối lập. Hầu hết các hệ thống reference counting cuối cùng cũng phải làm một chút tracing để xử lý vòng lặp, và các write barrier của một generational collector nếu nhìn kỹ cũng hơi giống các lệnh retain.

Để tìm hiểu sâu hơn, hãy xem “[A Unified Theory of Garbage Collection](https://researcher.watson.ibm.com/researcher/files/us-bacon/Bacon04Unified.pdf)” (PDF).

</aside>

Tracing garbage collection có tiếng là “đáng sợ”. Quả thật, làm việc ở mức bộ nhớ thô cũng hơi rùng mình. Debug một GC đôi khi khiến bạn thấy cả các bản dump hex trong giấc mơ. Nhưng hãy nhớ, cuốn sách này là để xua tan “ma thuật” và hạ gục những con quái vật đó, nên chúng ta *sẽ* tự viết một garbage collector. Tôi nghĩ bạn sẽ thấy thuật toán này khá đơn giản và rất thú vị để cài đặt.

## Data Types

Trong “vũ trụ” nhỏ bé của Lox, những nguyên tử tạo nên mọi thứ chính là các kiểu dữ liệu built-in. Chỉ có vài loại thôi:

*   **<span name="bool">Booleans</span>.** Bạn không thể lập trình nếu thiếu logic, và cũng không thể có logic nếu thiếu Boolean. “True” và “false” — âm và dương của phần mềm. Không giống một số ngôn ngữ cổ xưa tái sử dụng một kiểu dữ liệu có sẵn để biểu diễn đúng/sai, Lox có một kiểu Boolean riêng biệt. Chúng ta có thể đang “dã chiến” trong chuyến phiêu lưu này, nhưng không phải *mọi rợ*.

    <aside name="bool">

    Boolean là kiểu dữ liệu duy nhất trong Lox được đặt theo tên một người — George Boole — nên “Boolean” được viết hoa. Ông mất năm 1864, gần một thế kỷ trước khi máy tính số biến đại số của ông thành điện tử. Tôi tự hỏi nếu ông thấy tên mình xuất hiện trong hàng tỷ dòng code Java thì sẽ nghĩ gì.

    </aside>

    Có hai giá trị Boolean, tất nhiên rồi, và mỗi giá trị có một literal riêng.

    ```lox
    true;  // Not false.
    false; // Not *not* false.
    ```

*   **Numbers.** Lox chỉ có một loại số: số thực dấu chấm động double-precision. Vì số thực dấu chấm động cũng có thể biểu diễn một dải rộng các số nguyên, nên nó bao quát khá nhiều trường hợp, đồng thời giữ mọi thứ đơn giản.

    Các ngôn ngữ đầy đủ tính năng thường có nhiều cú pháp cho số — hệ thập lục phân, ký hiệu khoa học, bát phân, đủ kiểu thú vị. Chúng ta sẽ chỉ dùng literal số nguyên và số thập phân cơ bản.

    ```lox
    1234;  // An integer.
    12.34; // A decimal number.
    ```

*   **Strings.** Chúng ta đã thấy một string literal trong ví dụ đầu tiên. Giống hầu hết các ngôn ngữ, chúng được đặt trong dấu ngoặc kép.

    ```lox
    "I am a string";
    "";    // The empty string.
    "123"; // This is a string, not a number.
    ```

    Như chúng ta sẽ thấy khi cài đặt, có khá nhiều phức tạp ẩn sau chuỗi <span name="char">character</span> tưởng chừng vô hại này.

    <aside name="char">

    Ngay cả từ “character” cũng là một kẻ đánh lừa. Nó là ASCII? Unicode? Một code point hay một “grapheme cluster”? Các ký tự được mã hóa thế nào? Mỗi ký tự có kích thước cố định hay thay đổi?

    </aside>

*   **Nil.** Có một giá trị built-in cuối cùng, chẳng bao giờ được mời nhưng luôn xuất hiện. Nó biểu diễn “không có giá trị”. Trong nhiều ngôn ngữ khác, nó được gọi là “null”. Trong Lox, ta viết là `nil`. (Khi cài đặt, điều này sẽ giúp phân biệt khi nào ta nói về `nil` của Lox so với `null` của Java hoặc C.)

    Có nhiều lý lẽ để loại bỏ null khỏi một ngôn ngữ, vì lỗi null pointer là “tai họa” của ngành. Nếu chúng ta làm một ngôn ngữ static typing, có lẽ sẽ đáng để thử cấm nó. Nhưng trong một ngôn ngữ dynamic typing, loại bỏ nó thường gây phiền toái hơn là giữ lại.

## Expressions

Nếu các kiểu dữ liệu built-in và literal của chúng là nguyên tử, thì **expressions** chính là các phân tử. Phần lớn trong số này sẽ quen thuộc với bạn.

### Arithmetic

Lox có các toán tử số học cơ bản mà bạn biết và yêu thích từ C và các ngôn ngữ khác:

```lox
add + me;
subtract - me;
multiply * me;
divide / me;
```

Các biểu thức con ở hai bên toán tử gọi là **operands**. Vì có *hai* toán hạng, nên chúng được gọi là toán tử **binary**. (Điều này không liên quan gì đến nghĩa “nhị phân” kiểu 0 và 1.) Vì toán tử được <span name="fixity">cố định</span> *ở giữa* các toán hạng, nên chúng còn được gọi là toán tử **infix** (trái ngược với **prefix** — toán tử đứng trước toán hạng, và **postfix** — toán tử đứng sau).

<aside name="fixity">

Có một số toán tử có nhiều hơn hai toán hạng và toán tử được xen kẽ giữa chúng. Toán tử duy nhất được dùng rộng rãi là toán tử “điều kiện” hay “ternary” của C và các ngôn ngữ họ hàng:

```c
condition ? thenArm : elseArm;
```

Một số người gọi chúng là toán tử **mixfix**. Một vài ngôn ngữ cho phép bạn định nghĩa toán tử riêng và điều khiển vị trí của chúng — tức “fixity”.

</aside>

Một toán tử số học thực ra vừa là infix vừa là prefix: dấu `-` cũng có thể dùng để lấy số đối.

```lox
-negateMe;
```

Tất cả các toán tử này hoạt động trên số, và sẽ báo lỗi nếu truyền kiểu khác. Ngoại lệ duy nhất là toán tử `+` — bạn cũng có thể truyền cho nó hai chuỗi để nối chúng lại.

### Comparison and equality

Tiếp tục nào, chúng ta có thêm vài toán tử nữa luôn trả về kết quả Boolean.  
Ta có thể so sánh các số (và **chỉ** số) bằng những “toán tử so sánh cổ điển”:

```lox
less < than;
lessThan <= orEqual;
greater > than;
greaterThan >= orEqual;
```

Ta có thể kiểm tra hai giá trị bất kỳ xem chúng bằng nhau hay khác nhau:

```lox
1 == 2;         // false.
"cat" != "dog"; // true.
```

Thậm chí là khác kiểu:

```lox
314 == "pi"; // false.
```

Các giá trị khác kiểu *không bao giờ* được coi là tương đương:

```lox
123 == "123"; // false.
```

Nói chung, tôi không thích các phép chuyển đổi ngầm định.

### Logical operators

Toán tử not, dạng prefix `!`, trả về `false` nếu toán hạng của nó là true, và ngược lại.

```lox
!true;  // false.
!false; // true.
```

Hai toán tử logic còn lại thực chất là các cấu trúc điều khiển trá hình dưới dạng biểu thức.  
Biểu thức <span name="and">`and`</span> kiểm tra xem *cả hai* giá trị có đều true hay không. Nó trả về toán hạng bên trái nếu toán hạng đó là false, hoặc trả về toán hạng bên phải nếu không.

```lox
true and false; // false.
true and true;  // true.
```

Còn biểu thức `or` kiểm tra xem *ít nhất một* trong hai giá trị (hoặc cả hai) có true hay không. Nó trả về toán hạng bên trái nếu toán hạng đó là true, và trả về toán hạng bên phải nếu không.

```lox
false or false; // false.
true or false;  // true.
```

<aside name="and">

Tôi dùng `and` và `or` thay vì `&&` và `||` vì Lox không dùng `&` và `|` cho các toán tử bitwise. Sẽ thấy kỳ cục nếu đưa vào dạng hai ký tự mà không có dạng một ký tự.

Tôi cũng khá thích dùng từ khóa cho chúng vì thực chất đây là các cấu trúc điều khiển luồng, không chỉ là toán tử đơn giản.

</aside>

Lý do `and` và `or` giống cấu trúc điều khiển là vì chúng **short-circuit**. Không chỉ `and` trả về toán hạng bên trái nếu nó là false, mà trong trường hợp đó nó còn *không thèm* đánh giá toán hạng bên phải. Ngược lại (hay nói theo kiểu phản chứng?), nếu toán hạng bên trái của `or` là true, toán hạng bên phải sẽ bị bỏ qua.

### Precedence and grouping

Tất cả các toán tử này có độ ưu tiên (precedence) và tính kết hợp (associativity) giống như bạn mong đợi từ C. (Khi đến phần parsing, chúng ta sẽ nói *cực kỳ* chi tiết về chuyện này.) Trong những trường hợp độ ưu tiên không như ý, bạn có thể dùng `()` để nhóm lại.

```lox
var average = (min + max) / 2;
```

Vì chúng không quá thú vị về mặt kỹ thuật, tôi đã lược bỏ phần còn lại của “bộ sưu tập” toán tử thường thấy khỏi ngôn ngữ nhỏ bé của chúng ta. Không có bitwise, shift, modulo hay toán tử điều kiện. Tôi không chấm điểm bạn, nhưng bạn sẽ được cộng điểm trong lòng tôi nếu tự bổ sung chúng vào bản cài đặt Lox của mình.

Đó là các dạng expression (trừ một vài dạng liên quan đến các tính năng cụ thể mà ta sẽ gặp sau), nên giờ hãy lên một cấp độ.

## Statements

Giờ chúng ta đến với statements. Nếu nhiệm vụ chính của một expression là tạo ra một *giá trị*, thì nhiệm vụ của một statement là tạo ra một *tác động*. Vì theo định nghĩa, statements không trả về giá trị, nên để hữu ích, chúng phải thay đổi “thế giới” theo cách nào đó — thường là thay đổi trạng thái, đọc input hoặc tạo output.

Bạn đã thấy vài loại statement rồi. Loại đầu tiên là:

```lox
print "Hello, world!";
```

Một <span name="print">`print` statement</span> sẽ đánh giá một expression duy nhất và hiển thị kết quả cho người dùng. Bạn cũng đã thấy vài statement như:

<aside name="print">

Đưa `print` vào thẳng ngôn ngữ thay vì chỉ làm nó thành một hàm trong core library là một “mánh”. Nhưng đây là một mánh *hữu ích* cho chúng ta: nó cho phép interpreter đang xây dựng có thể tạo ra output trước khi chúng ta cài đặt toàn bộ cơ chế để định nghĩa hàm, tìm chúng theo tên và gọi chúng.

</aside>

```lox
"some expression";
```

Một expression theo sau bởi dấu chấm phẩy (`;`) sẽ được “nâng cấp” thành một statement. Cái này được gọi (một cách khá… tưởng tượng) là **expression statement**.

Nếu bạn muốn gói một loạt statement vào chỗ mà chỉ một statement được mong đợi, bạn có thể bọc chúng trong một **block**.

```lox
{
  print "One statement.";
  print "Two statements.";
}
```

Block cũng ảnh hưởng đến phạm vi (scope), và điều đó dẫn chúng ta đến phần tiếp theo...

## Variables

Bạn khai báo biến bằng `var` statement. Nếu bạn <span name="omit">bỏ qua</span> phần khởi tạo, giá trị của biến sẽ mặc định là `nil`.

<aside name="omit">

Đây là một trong những trường hợp mà việc không có `nil` và buộc mọi biến phải được khởi tạo với một giá trị nào đó sẽ gây phiền toái hơn là việc chấp nhận `nil`.

</aside>

```lox
var imAVariable = "here is my value";
var iAmNil;
```

Sau khi khai báo, tất nhiên, bạn có thể truy cập và gán giá trị cho biến bằng tên của nó.

<span name="breakfast"></span>

```lox
var breakfast = "bagels";
print breakfast; // "bagels".
breakfast = "beignets";
print breakfast; // "beignets".
```

<aside name="breakfast">

Bạn có nhận ra là tôi thường làm việc với cuốn sách này vào buổi sáng trước khi ăn gì không?

</aside>

Tôi sẽ không đi sâu vào các quy tắc về phạm vi biến ở đây, vì chúng ta sẽ dành khá nhiều thời gian ở các chương sau để “vẽ bản đồ” từng ngóc ngách của các quy tắc đó. Trong hầu hết các trường hợp, nó hoạt động giống như bạn mong đợi khi đến từ C hoặc Java.

## Control Flow

Thật khó để viết các chương trình <span name="flow">hữu ích</span> nếu bạn không thể bỏ qua một số đoạn code hoặc execute một số đoạn nhiều lần. Đó chính là control flow. Ngoài các toán tử logic mà ta đã nói, Lox lấy nguyên ba loại statement từ C.

<aside name="flow">

Chúng ta đã có `and` và `or` để rẽ nhánh, và *có thể* dùng đệ quy để lặp lại code, nên về lý thuyết là đủ. Nhưng lập trình theo kiểu đó trong một ngôn ngữ mang phong cách mệnh lệnh sẽ khá gượng gạo.

Scheme, ngược lại, không có cấu trúc vòng lặp built-in. Nó *thực sự* dựa vào đệ quy để lặp. Smalltalk thì không có cấu trúc rẽ nhánh built-in, và dựa vào dynamic dispatch để chọn lọc execute code.

</aside>

Một `if` statement sẽ execute một trong hai statement dựa trên điều kiện.

```lox
if (condition) {
  print "yes";
} else {
  print "no";
}
```

Một vòng lặp `while` <span name="do">loop</span> sẽ execute phần thân lặp đi lặp lại miễn là biểu thức điều kiện vẫn true.

```lox
var a = 1;
while (a < 10) {
  print a;
  a = a + 1;
}
```

<aside name="do">

Tôi bỏ `do while` loop ra khỏi Lox vì chúng không phổ biến lắm và cũng không dạy bạn điều gì mà bạn chưa học được từ `while`. Cứ thoải mái thêm nó vào bản cài đặt của bạn nếu bạn thích. Đây là “bữa tiệc” của bạn mà.

</aside>

Cuối cùng, chúng ta có vòng lặp `for`.

```lox
for (var a = 1; a < 10; a = a + 1) {
  print a;
}
```

Vòng lặp này làm đúng những gì vòng `while` trước đó làm. Hầu hết các ngôn ngữ hiện đại cũng có dạng vòng lặp <span name="foreach">`for-in`</span> hoặc `foreach` để duyệt qua các kiểu sequence khác nhau một cách rõ ràng. Trong một ngôn ngữ thực tế, điều đó “dễ chịu” hơn nhiều so với vòng `for` kiểu C thô sơ mà ta có ở đây. Lox thì giữ mọi thứ đơn giản.

<aside name="foreach">

Đây là một sự nhượng bộ tôi đưa ra vì cách mà phần cài đặt được chia theo chương. Một vòng `for-in` cần một dạng dynamic dispatch trong iterator protocol để xử lý các loại sequence khác nhau, nhưng chúng ta sẽ không có điều đó cho đến khi xong phần control flow. Chúng ta có thể quay lại và thêm vòng `for-in` sau, nhưng tôi không nghĩ việc đó sẽ dạy bạn điều gì quá thú vị.

</aside>

## Functions

Một biểu thức gọi hàm (function call expression) trong Lox trông giống hệt như trong C.

```lox
makeBreakfast(bacon, eggs, toast);
```

Bạn cũng có thể gọi một hàm mà không truyền gì vào.

```lox
makeBreakfast();
```

Không giống như trong, chẳng hạn, Ruby, dấu ngoặc đơn là bắt buộc trong trường hợp này. Nếu bạn bỏ chúng đi, tên hàm sẽ không *gọi* hàm đó, mà chỉ đơn thuần tham chiếu tới nó.

Một ngôn ngữ sẽ chẳng thú vị mấy nếu bạn không thể tự định nghĩa hàm của mình. Trong Lox, bạn làm điều đó với <span name="fun">`fun`</span>.

<aside name="fun">

Tôi đã thấy các ngôn ngữ dùng `fn`, `fun`, `func`, và `function`. Tôi vẫn hy vọng sẽ tìm thấy đâu đó một `funct`, `functi`, hoặc `functio`.

</aside>

```lox
fun printSum(a, b) {
  print a + b;
}
```

Giờ là lúc thích hợp để làm rõ một chút về <span name="define">thuật ngữ</span>. Một số người dùng “parameter” và “argument” như thể chúng có thể thay thế cho nhau, và với nhiều người thì đúng là vậy. Nhưng chúng ta sẽ dành khá nhiều thời gian để mổ xẻ những chi tiết nhỏ nhất về ngữ nghĩa, nên hãy làm rõ từ ngữ. Từ đây trở đi:

*   **Argument** là giá trị thực tế bạn truyền vào hàm khi gọi nó. Vì vậy, một lần *gọi* hàm sẽ có một danh sách *argument*. Đôi khi bạn sẽ nghe thấy cụm **actual parameter** để chỉ chúng.

*   **Parameter** là biến giữ giá trị của argument bên trong thân hàm. Do đó, một *khai báo* hàm sẽ có một danh sách *parameter*. Một số người gọi chúng là **formal parameters** hoặc đơn giản là **formals**.

<aside name="define">

Nói về thuật ngữ, một số ngôn ngữ static typing như C phân biệt giữa *khai báo* (declare) hàm và *định nghĩa* (define) hàm. Khai báo sẽ gắn kiểu của hàm với tên của nó để các lời gọi có thể được kiểm tra kiểu, nhưng không cung cấp phần thân. Định nghĩa vừa khai báo hàm vừa viết phần thân để hàm có thể được biên dịch.

Vì Lox là dynamic typing, sự phân biệt này không có ý nghĩa. Một khai báo hàm trong Lox đã bao gồm đầy đủ phần thân.

</aside>

Phần thân của một hàm luôn là một block. Bên trong, bạn có thể trả về một giá trị bằng `return` statement.

```lox
fun returnSum(a, b) {
  return a + b;
}
```

Nếu việc execute đi đến cuối block mà không gặp `return`, nó sẽ <span name="sneaky">ngầm định</span> trả về `nil`.

<aside name="sneaky">

Thấy chưa, tôi đã nói là `nil` sẽ lén xuất hiện khi ta không để ý mà.

</aside>

### Closures

Hàm trong Lox là *first class*, nghĩa là chúng là những giá trị thực sự mà bạn có thể lấy tham chiếu, lưu vào biến, truyền đi, v.v. Ví dụ này hoạt động:

```lox
fun addPair(a, b) {
  return a + b;
}

fun identity(a) {
  return a;
}

print identity(addPair)(1, 2); // Prints "3".
```

Vì khai báo hàm là một statement, bạn có thể khai báo hàm cục bộ bên trong một hàm khác.

```lox
fun outerFunction() {
  fun localFunction() {
    print "I'm local!";
  }

  localFunction();
}
```

Nếu bạn kết hợp hàm cục bộ, hàm first-class và block scope, bạn sẽ gặp tình huống thú vị này:

```lox
fun returnFunction() {
  var outside = "outside";

  fun inner() {
    print outside;
  }

  return inner;
}

var fn = returnFunction();
fn();
```

Ở đây, `inner()` truy cập một biến cục bộ được khai báo bên ngoài thân của nó, trong hàm bao quanh. Điều này có hợp lệ không? Giờ khi nhiều ngôn ngữ đã mượn tính năng này từ Lisp, có lẽ bạn đã biết câu trả lời là “có”.

Để làm được điều đó, `inner()` phải “giữ lại” tham chiếu tới bất kỳ biến bao quanh nào mà nó sử dụng, để chúng vẫn tồn tại ngay cả sau khi hàm bên ngoài đã trả về. Chúng ta gọi những hàm làm điều này là <span name="closure">**closures**</span>. Ngày nay, thuật ngữ này thường được dùng cho *bất kỳ* hàm first-class nào, dù thực ra hơi sai nếu hàm đó không “đóng” (close over) biến nào cả.

<aside name="closure">

Peter J. Landin là người đặt ra thuật ngữ "closure". Vâng, ông ấy đã phát minh ra gần như một nửa số thuật ngữ trong lĩnh vực ngôn ngữ lập trình. Phần lớn trong số đó xuất phát từ một bài báo kinh điển, “[The Next 700 Programming Languages](https://homepages.inf.ed.ac.uk/wadler/papers/papers-we-love/landin-next-700.pdf)”.

Để cài đặt những loại hàm này, bạn cần tạo ra một cấu trúc dữ liệu gói chung cả phần code của hàm và các biến bao quanh mà nó cần. Ông gọi nó là "closure" vì nó *đóng gói* (close over) và giữ lại các biến cần thiết.

</aside>

Như bạn có thể hình dung, việc cài đặt chúng sẽ làm tăng độ phức tạp vì ta không thể giả định rằng phạm vi biến hoạt động hoàn toàn như một stack — nơi các biến cục bộ biến mất ngay khi hàm trả về. Chúng ta sẽ có một khoảng thời gian thú vị để học cách làm cho chúng hoạt động đúng và hiệu quả.

## Classes

Vì Lox có dynamic typing, lexical scope (nôm na là “block” scope) và closures, nó đã đi được nửa chặng đường để trở thành một ngôn ngữ lập trình hàm (functional language). Nhưng như bạn sẽ thấy, nó *cũng* đã đi được nửa chặng đường để trở thành một ngôn ngữ hướng đối tượng (object-oriented language). Cả hai mô hình đều có nhiều điểm mạnh, nên tôi nghĩ đáng để đề cập một chút về mỗi bên.

Vì classes gần đây bị chỉ trích là không đáp ứng được kỳ vọng, trước tiên hãy để tôi giải thích tại sao tôi đưa chúng vào Lox và vào cuốn sách này. Thực ra có hai câu hỏi:

### Tại sao một ngôn ngữ lại muốn hướng đối tượng?

Giờ đây, khi các ngôn ngữ hướng đối tượng như Java đã “bán hết vé” và chỉ diễn ở những sân khấu lớn, việc thích chúng không còn “ngầu” nữa. Vậy tại sao ai đó lại tạo ra một ngôn ngữ *mới* với objects? Chẳng phải điều đó giống như phát hành nhạc trên băng 8-track sao?

Đúng là “cơn sốt kế thừa mọi thứ” của thập niên 90 đã tạo ra những hệ thống class kế thừa khổng lồ và cồng kềnh, nhưng **lập trình hướng đối tượng** (**OOP**) vẫn rất tuyệt. Hàng tỷ dòng code thành công đã được viết bằng các ngôn ngữ OOP, đưa hàng triệu ứng dụng đến tay người dùng hài lòng. Có lẽ phần lớn lập trình viên hiện nay đang dùng một ngôn ngữ hướng đối tượng. Không thể tất cả họ đều *sai* được.

Đặc biệt, với một ngôn ngữ dynamic typing, objects rất hữu ích. Chúng ta cần *một cách nào đó* để định nghĩa các kiểu dữ liệu phức hợp nhằm gom nhiều thứ lại với nhau.

Nếu ta có thể gắn thêm methods vào đó, thì ta tránh được việc phải đặt tiền tố cho tất cả các hàm bằng tên kiểu dữ liệu mà chúng xử lý, nhằm tránh xung đột với các hàm tương tự cho kiểu khác. Ví dụ, trong Racket, bạn sẽ phải đặt tên hàm như `hash-copy` (để copy một hash table) và `vector-copy` (để copy một vector) để chúng không “dẫm chân” nhau. Methods được giới hạn phạm vi trong object, nên vấn đề đó biến mất.

### Tại sao Lox lại hướng đối tượng?

Tôi có thể nói rằng objects rất “ngầu” nhưng vẫn nằm ngoài phạm vi của cuốn sách. Hầu hết sách về ngôn ngữ lập trình, đặc biệt là những cuốn cố gắng cài đặt cả một ngôn ngữ, đều bỏ qua objects. Với tôi, điều đó có nghĩa là chủ đề này chưa được đề cập đầy đủ. Với một mô hình lập trình phổ biến như vậy, việc bỏ qua khiến tôi thấy tiếc.

Xét việc nhiều người trong chúng ta dành cả ngày *sử dụng* các ngôn ngữ OOP, có vẻ thế giới sẽ cần một chút tài liệu về cách *tạo ra* một ngôn ngữ như vậy. Như bạn sẽ thấy, hóa ra nó khá thú vị. Không khó như bạn lo, nhưng cũng không đơn giản như bạn tưởng.

### Classes hay prototypes

Khi nói đến objects, thực ra có hai cách tiếp cận: [classes](https://en.wikipedia.org/wiki/Class-based_programming) và [prototypes](https://en.wikipedia.org/wiki/Prototype-based_programming). Classes xuất hiện trước và phổ biến hơn nhờ C++, Java, C# và các “anh em” của chúng. Prototypes từng là một nhánh gần như bị lãng quên cho đến khi JavaScript vô tình “thống trị thế giới”.

Trong các ngôn ngữ dựa trên class, có hai khái niệm cốt lõi: instance và class. Instance lưu trữ trạng thái cho từng object và có tham chiếu đến class của instance đó. Class chứa các method và chuỗi kế thừa. Để gọi một method trên một instance, luôn có một bước gián tiếp. Bạn sẽ <span name="dispatch">tra cứu</span> class của instance đó rồi tìm method *ở đó*:

<aside name="dispatch">

Trong một ngôn ngữ static typing như C++, việc tìm method thường diễn ra ở thời điểm biên dịch dựa trên kiểu *tĩnh* của instance, cho bạn **static dispatch**. Ngược lại, **dynamic dispatch** sẽ tra cứu class của object thực tế tại runtime. Đây là cách các virtual method trong ngôn ngữ static typing và tất cả method trong ngôn ngữ dynamic typing như Lox hoạt động.

</aside>

<img src="image/the-lox-language/class-lookup.png" alt="Cách tra cứu fields và methods trên classes và instances" />

Các ngôn ngữ dựa trên prototype <span name="blurry">kết hợp</span> hai khái niệm này. Chỉ có objects — không có classes — và mỗi object riêng lẻ có thể chứa cả trạng thái và methods. Objects có thể kế thừa trực tiếp từ nhau (hoặc “ủy quyền” trong ngôn ngữ của prototype):

<aside name="blurry">

Trên thực tế, ranh giới giữa ngôn ngữ dựa trên class và dựa trên prototype khá mờ. Khái niệm “constructor function” của JavaScript [đẩy bạn khá mạnh](http://gameprogrammingpatterns.com/prototype.html#what-about-javascript) theo hướng định nghĩa các object giống class. Trong khi đó, Ruby dựa trên class lại hoàn toàn thoải mái cho phép bạn gắn methods vào từng instance riêng lẻ.

</aside>

<img src="image/the-lox-language/prototype-lookup.png" alt="How fields and methods are looked up in a prototypal system" />

Điều này có nghĩa là, theo một số khía cạnh, các ngôn ngữ dựa trên prototype mang tính nền tảng hơn so với classes. Chúng thực sự rất “đã” khi cài đặt vì *quá* đơn giản. Ngoài ra, chúng có thể biểu đạt nhiều mẫu thiết kế (pattern) khác thường mà classes thường hướng bạn tránh xa.

Nhưng tôi đã xem *rất nhiều* code viết bằng các ngôn ngữ dựa trên prototype — bao gồm cả [một số do chính tôi tạo ra](http://finch.stuffwithstuff.com/). Bạn có biết mọi người thường làm gì với tất cả sức mạnh và sự linh hoạt của prototypes không? …Họ dùng chúng để tái tạo lại classes.

Tôi không biết *tại sao* lại như vậy, nhưng dường như con người tự nhiên có xu hướng thích phong cách dựa trên class (Classic? Classy?). Prototypes *đúng là* đơn giản hơn trong ngôn ngữ, nhưng chúng dường như đạt được điều đó chỉ bằng cách <span name="waterbed">đẩy</span> phần phức tạp sang cho người dùng. Vậy nên, với Lox, chúng ta sẽ giúp người dùng đỡ vất vả và tích hợp sẵn classes vào.

<aside name="waterbed">

Larry Wall, “cha đẻ” kiêm “nhà tiên tri” của Perl, gọi đây là “[waterbed theory](http://wiki.c2.com/?WaterbedTheory)”. Một số sự phức tạp là thiết yếu và không thể loại bỏ. Nếu bạn ấn nó xuống ở chỗ này, nó sẽ phồng lên ở chỗ khác.

Các ngôn ngữ dựa trên prototype không hẳn *loại bỏ* sự phức tạp của classes, mà là bắt *người dùng* gánh lấy sự phức tạp đó bằng cách tự xây dựng các thư viện metaprogramming giống class.

</aside>

### Classes trong Lox

Bấy nhiêu lý do là đủ rồi, giờ hãy xem chúng ta thực sự có gì. Trong hầu hết các ngôn ngữ, classes bao gồm một “chòm sao” các tính năng. Với Lox, tôi đã chọn ra những “ngôi sao” sáng nhất. Bạn khai báo một class và các method của nó như sau:

```lox
class Breakfast {
  cook() {
    print "Eggs a-fryin'!";
  }

  serve(who) {
    print "Enjoy your breakfast, " + who + ".";
  }
}
```

Phần thân của class chứa các method. Chúng trông giống như khai báo hàm nhưng không có từ khóa `fun` <span name="method">keyword</span>. Khi câu lệnh khai báo class được execute, Lox sẽ tạo ra một object class và lưu nó vào một biến có tên trùng với tên class. Giống như functions, classes cũng là first-class trong Lox.

<aside name="method">

Nhưng chúng vẫn “fun” như thường.

</aside>

```lox
// Lưu vào biến.
var someVariable = Breakfast;

// Truyền vào hàm.
someFunction(Breakfast);
```

Tiếp theo, chúng ta cần một cách để tạo instance. Ta có thể thêm một từ khóa `new`, nhưng để giữ mọi thứ đơn giản, trong Lox, bản thân class đóng vai trò như một factory function cho các instance. Gọi class như gọi một hàm, và nó sẽ tạo ra một instance mới của chính nó.

```lox
var breakfast = Breakfast();
print breakfast; // "Breakfast instance".
```

### Khởi tạo (Instantiation) & initialization

Các class chỉ có hành vi (behavior) thì không quá hữu ích. Ý tưởng đằng sau lập trình hướng đối tượng là đóng gói cả hành vi *và trạng thái* (state) lại với nhau. Để làm được điều đó, bạn cần có fields. Lox, giống như các ngôn ngữ dynamic typing khác, cho phép bạn tự do thêm thuộc tính (property) vào objects.

```lox
breakfast.meat = "sausage";
breakfast.bread = "sourdough";
```

Gán giá trị cho một field sẽ tạo field đó nếu nó chưa tồn tại.

Nếu bạn muốn truy cập một field hoặc method trên object hiện tại từ bên trong một method, bạn dùng `this` quen thuộc.

```lox
class Breakfast {
  serve(who) {
    print "Enjoy your " + this.meat + " and " +
        this.bread + ", " + who + ".";
  }

  // ...
}
```

Một phần của việc đóng gói dữ liệu trong object là đảm bảo object ở trạng thái hợp lệ khi được tạo ra. Để làm điều đó, bạn có thể định nghĩa một initializer. Nếu class của bạn có một method tên `init()`, nó sẽ được gọi tự động khi object được khởi tạo. Bất kỳ tham số nào truyền vào class sẽ được chuyển tiếp đến initializer.

```lox
class Breakfast {
  init(meat, bread) {
    this.meat = meat;
    this.bread = bread;
  }

  // ...
}

var baconAndToast = Breakfast("bacon", "toast");
baconAndToast.serve("Dear Reader");
// "Enjoy your bacon and toast, Dear Reader."
```

### Inheritance

Mọi ngôn ngữ hướng đối tượng đều cho phép bạn không chỉ định nghĩa methods, mà còn tái sử dụng chúng giữa nhiều class hoặc object. Để làm điều đó, Lox hỗ trợ single inheritance. Khi bạn khai báo một class, bạn có thể chỉ định class mà nó kế thừa bằng toán tử nhỏ hơn <span name="less">(`<`)</span>.

```lox
class Brunch < Breakfast {
  drink() {
    print "How about a Bloody Mary?";
  }
}
```

<aside name="less">

Tại sao lại dùng toán tử `<`? Tôi không muốn giới thiệu thêm một từ khóa mới như `extends`. Lox cũng không dùng `:` cho thứ gì khác nên tôi cũng không muốn “giữ chỗ” cho nó. Thay vào đó, tôi học theo Ruby và dùng `<`.

Nếu bạn biết một chút về lý thuyết kiểu (type theory), bạn sẽ nhận ra đây không phải là một lựa chọn *hoàn toàn* tùy tiện. Mọi instance của subclass cũng là instance của superclass, nhưng có thể có những instance của superclass không phải là instance của subclass. Điều đó có nghĩa là, trong “vũ trụ” các object, tập hợp các object thuộc subclass nhỏ hơn tập hợp của superclass, dù dân “mọt” kiểu thường dùng ký hiệu `<:` cho quan hệ này.

</aside>

Ở đây, Brunch là **derived class** hay **subclass**, và Breakfast là **base class** hay **superclass**.

Mọi method được định nghĩa trong superclass đều có sẵn cho các subclass.

```lox
var benedict = Brunch("ham", "English muffin");
benedict.serve("Noble Reader");
```

Ngay cả method `init()` cũng được <span name="init">kế thừa</span>. Trên thực tế, subclass thường muốn định nghĩa method `init()` của riêng mình. Nhưng method gốc cũng cần được gọi để superclass có thể duy trì trạng thái của nó. Chúng ta cần một cách để gọi method trên *instance* của chính mình mà không “đụng” vào method của chính mình.

<aside name="init">

Lox khác với C++, Java và C#, vốn không kế thừa constructors, nhưng giống với Smalltalk và Ruby, vốn có kế thừa.

</aside>

Giống như trong Java, bạn dùng `super` cho việc đó.

```lox
class Brunch < Breakfast {
  init(meat, bread, drink) {
    super.init(meat, bread);
    this.drink = drink;
  }
}
```

Về cơ bản, đó là tất cả về lập trình hướng đối tượng trong Lox. Tôi cố gắng giữ bộ tính năng ở mức tối giản. Cấu trúc của cuốn sách buộc tôi phải chấp nhận một điểm thỏa hiệp: Lox không phải là một ngôn ngữ hướng đối tượng *thuần túy*. Trong một ngôn ngữ OOP “thật sự”, mọi object đều là instance của một class, kể cả các giá trị nguyên thủy như số và Boolean.

Bởi vì chúng ta không cài đặt classes cho đến khá lâu sau khi đã làm việc với các kiểu built-in, nên điều đó sẽ rất khó. Vì vậy, các giá trị kiểu nguyên thủy không phải là object “thật” theo nghĩa là instance của class. Chúng không có methods hay properties. Nếu tôi định biến Lox thành một ngôn ngữ thực sự cho người dùng thực sự, tôi sẽ sửa điều đó.

## The Standard Library

Chúng ta gần xong rồi. Đó là toàn bộ ngôn ngữ, nên phần còn lại chỉ là “core” hay “standard” library — tập hợp các chức năng được cài đặt trực tiếp trong interpreter và là nền tảng cho mọi hành vi do người dùng định nghĩa.

Đây là phần “buồn” nhất của Lox. Standard library của nó vượt xa mức tối giản và gần như chạm tới chủ nghĩa hư vô. Với các ví dụ trong sách, chúng ta chỉ cần chứng minh rằng code đang chạy và làm đúng những gì nó cần làm. Để làm điều đó, chúng ta đã có sẵn câu lệnh `print` built-in.

Sau này, khi bắt đầu tối ưu hóa, chúng ta sẽ viết một số benchmark và xem mất bao lâu để execute code. Điều đó có nghĩa là chúng ta cần theo dõi thời gian, nên sẽ định nghĩa một hàm built-in, `clock()`, trả về số giây kể từ khi chương trình bắt đầu chạy.

Và… hết rồi. Tôi biết, nghe thật xấu hổ.

Nếu bạn muốn biến Lox thành một ngôn ngữ thực sự hữu ích, việc đầu tiên bạn nên làm là mở rộng phần này. Xử lý chuỗi, các hàm lượng giác, file I/O, networking, thậm chí *đọc input từ người dùng* cũng sẽ giúp ích. Nhưng chúng ta không cần những thứ đó cho cuốn sách này, và thêm chúng vào cũng không dạy bạn điều gì thú vị, nên tôi đã bỏ qua.

Đừng lo, bản thân ngôn ngữ sẽ có đủ thứ thú vị để giữ chúng ta bận rộn.


<div class="challenges">

## Thử thách

1. Viết một vài chương trình Lox mẫu và chạy chúng (bạn có thể dùng các bản cài đặt Lox trong [repository của tôi](https://github.com/munificent/craftinginterpreters)). Hãy thử nghĩ ra các trường hợp “edge case” mà tôi chưa chỉ rõ ở đây. Nó có hoạt động như bạn mong đợi không? Tại sao có hoặc tại sao không?

2. Phần giới thiệu không chính thức này bỏ ngỏ *rất nhiều* chi tiết. Hãy liệt kê một số câu hỏi mở mà bạn có về cú pháp và ngữ nghĩa của ngôn ngữ. Bạn nghĩ câu trả lời nên như thế nào?

3. Lox là một ngôn ngữ khá nhỏ gọn. Bạn nghĩ nó thiếu những tính năng nào khiến việc dùng nó cho các chương trình thực tế trở nên bất tiện? (Tất nhiên là ngoại trừ standard library.)

</div>

<div class="design-note">

## Ghi chú thiết kế: Expressions & Statements

Lox có cả expressions và statements. Một số ngôn ngữ bỏ hẳn loại thứ hai. Thay vào đó, chúng coi cả khai báo và các cấu trúc điều khiển luồng cũng là expressions. Những ngôn ngữ “mọi thứ đều là expression” này thường có nguồn gốc lập trình hàm và bao gồm hầu hết các Lisp, SML, Haskell, Ruby và CoffeeScript.

Để làm được điều đó, với mỗi cấu trúc “giống statement” trong ngôn ngữ, bạn cần quyết định giá trị mà nó sẽ trả về. Một số trường hợp khá dễ:

*   Một `if` expression trả về kết quả của nhánh được chọn. Tương tự, một `switch` hoặc các cấu trúc rẽ nhánh nhiều hướng khác trả về kết quả của case được chọn.

*   Một khai báo biến trả về giá trị của biến đó.

*   Một block trả về kết quả của expression cuối cùng trong chuỗi.

Một số trường hợp thì kỳ lạ hơn. Vòng lặp nên trả về gì? Trong CoffeeScript, một vòng `while` trả về một mảng chứa từng giá trị mà phần thân vòng lặp trả về. Điều này có thể hữu ích, hoặc lãng phí bộ nhớ nếu bạn không cần mảng đó.

Bạn cũng phải quyết định cách các expression dạng statement này kết hợp với các expression khác — tức là phải đặt chúng vào bảng độ ưu tiên (precedence table) của grammar. Ví dụ, Ruby cho phép:

```ruby
puts 1 + if true then 2 else 3 end + 4
```

Đây có phải điều bạn mong đợi không? Có phải điều *người dùng* của bạn mong đợi không? Điều này ảnh hưởng thế nào đến cách bạn thiết kế cú pháp cho “statements” của mình? Lưu ý rằng Ruby có từ khóa `end` rõ ràng để đánh dấu khi nào `if` expression kết thúc. Nếu không có nó, `+ 4` rất có thể sẽ bị parse như một phần của mệnh đề `else`.

Chuyển mọi statement thành expression buộc bạn phải trả lời một vài câu hỏi “khó nhằn” như vậy. Đổi lại, bạn loại bỏ được một số sự trùng lặp. C có cả block để sắp xếp các statement, và toán tử dấu phẩy để sắp xếp các expression. Nó có cả `if` statement và toán tử điều kiện `?:`. Nếu mọi thứ trong C đều là expression, bạn có thể hợp nhất từng cặp này.

Các ngôn ngữ loại bỏ statements thường cũng có **implicit returns** — một hàm tự động trả về giá trị mà phần thân của nó đánh giá được mà không cần cú pháp `return` rõ ràng. Với các hàm và method nhỏ, điều này rất tiện. Thực tế, nhiều ngôn ngữ vốn có statements cũng đã thêm cú pháp như `=>` để định nghĩa các hàm mà phần thân chỉ là kết quả của một expression duy nhất.

Nhưng bắt *tất cả* hàm hoạt động theo cách đó có thể hơi kỳ lạ. Nếu không cẩn thận, hàm của bạn sẽ “rò rỉ” giá trị trả về ngay cả khi bạn chỉ định nó tạo ra side effect. Tuy nhiên, trên thực tế, người dùng các ngôn ngữ này thường không thấy đó là vấn đề.

Với Lox, tôi giữ lại statements vì lý do thực dụng. Tôi chọn cú pháp giống C để tạo sự quen thuộc, và việc cố gắng lấy cú pháp statement của C rồi diễn giải nó như expressions sẽ trở nên kỳ quặc rất nhanh.

</div>