> Truyện cổ tích còn thật hơn cả sự thật: không phải vì chúng nói với ta rằng rồng tồn tại, mà vì chúng nói với ta rằng rồng có thể bị đánh bại.
>
> <cite>G.K. Chesterton qua lời Neil Gaiman, <em>Coraline</em></cite>

Tôi thực sự rất hào hứng khi chúng ta cùng nhau bắt đầu hành trình này. Đây là một cuốn sách về cách hiện thực hóa interpreter cho ngôn ngữ lập trình. Nó cũng là một cuốn sách về cách thiết kế một ngôn ngữ đáng để hiện thực. Đây là cuốn sách mà tôi ước mình đã có khi mới bắt đầu tìm hiểu về ngôn ngữ, và cũng là cuốn sách mà tôi đã “viết” trong <span name="head">đầu</span> mình suốt gần một thập kỷ qua.

<aside name="head">

Gửi tới bạn bè và gia đình: xin lỗi vì tôi đã lơ đãng bấy lâu nay!

</aside>

Trong những trang sách này, chúng ta sẽ cùng nhau đi từng bước qua hai interpreter hoàn chỉnh cho một ngôn ngữ đầy đủ tính năng. Tôi giả định đây là lần đầu bạn dấn thân vào thế giới ngôn ngữ lập trình, nên tôi sẽ giải thích từng khái niệm và từng dòng code bạn cần để xây dựng một bản hiện thực ngôn ngữ hoàn chỉnh, có thể sử dụng và chạy nhanh.

Để nhét vừa hai bản hiện thực đầy đủ vào một cuốn sách mà không biến nó thành một “cục gạch”, phần lý thuyết ở đây sẽ nhẹ hơn so với các sách khác. Khi xây dựng từng phần của hệ thống, tôi sẽ giới thiệu lịch sử và các khái niệm đằng sau nó. Tôi sẽ cố gắng giúp bạn quen với thuật ngữ để nếu một ngày bạn lạc vào một <span name="party">bữa tiệc cocktail</span> toàn các nhà nghiên cứu PL (programming language), bạn vẫn có thể hòa nhập.

<aside name="party">

Thật kỳ lạ là tôi đã rơi vào tình huống đó nhiều lần. Bạn sẽ không tin được là một số người trong họ có thể uống nhiều đến mức nào đâu.

</aside>

Nhưng phần lớn thời gian, chúng ta sẽ dồn chất xám để khiến ngôn ngữ chạy được. Điều này không có nghĩa là lý thuyết không quan trọng. Khả năng suy luận chính xác và <span name="formal">chặt chẽ</span> về cú pháp và ngữ nghĩa là một kỹ năng thiết yếu khi làm việc với ngôn ngữ. Nhưng cá nhân tôi học tốt nhất bằng cách “xắn tay” làm. Tôi khó mà tiếp thu hết những đoạn văn đầy khái niệm trừu tượng. Nhưng nếu tôi đã code, chạy và debug nó, thì tôi *hiểu* nó.

<aside name="formal">

Đặc biệt, hệ thống kiểu tĩnh đòi hỏi khả năng suy luận hình thức nghiêm ngặt. Làm việc với một hệ thống kiểu mang lại cảm giác giống như đang chứng minh một định lý toán học.

Điều này không phải ngẫu nhiên. Nửa đầu thế kỷ trước, Haskell Curry và William Alvin Howard đã chỉ ra rằng chúng là hai mặt của cùng một đồng xu: [đẳng cấu Curry–Howard](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence).

</aside>

Đó là mục tiêu của tôi dành cho bạn. Tôi muốn bạn có được trực giác vững chắc về cách một ngôn ngữ thực sự “sống” và “thở”. Tôi hy vọng rằng khi bạn đọc những cuốn sách lý thuyết hơn sau này, các khái niệm trong đó sẽ bám chặt vào trí nhớ của bạn, gắn liền với nền tảng cụ thể này.

## Tại sao nên học những thứ này?

Hầu như cuốn sách compiler nào cũng có phần này ở phần mở đầu. Tôi không hiểu tại sao ngôn ngữ lập trình lại dễ khiến người ta hoài nghi về sự tồn tại của chính nó như vậy. Tôi không nghĩ sách về điểu học lại phải bận tâm biện minh cho sự tồn tại của mình — họ mặc định người đọc yêu chim và bắt đầu dạy thôi.

Nhưng ngôn ngữ lập trình thì hơi khác. Có lẽ đúng là khả năng mỗi chúng ta tạo ra một ngôn ngữ lập trình đa dụng, thành công rộng rãi là rất nhỏ. Những nhà thiết kế của các ngôn ngữ phổ biến nhất thế giới có thể ngồi vừa một chiếc xe Volkswagen bus, thậm chí không cần mở mui. Nếu gia nhập nhóm tinh hoa đó là *lý do duy nhất* để học về ngôn ngữ, thì thật khó để biện minh. May mắn thay, không phải vậy.

### “Little languages” ở khắp mọi nơi

Với mỗi ngôn ngữ đa dụng thành công, có hàng ngàn ngôn ngữ chuyên biệt thành công. Trước đây chúng ta gọi chúng là “little languages”, nhưng “lạm phát” trong giới thuật ngữ đã dẫn đến cái tên “domain-specific languages” (DSL). Đây là những “ngôn ngữ lai” được thiết kế riêng cho một nhiệm vụ cụ thể. Hãy nghĩ đến ngôn ngữ scripting cho ứng dụng, engine template, định dạng markup, và file cấu hình.

<span name="little"></span><img src="image/introduction/little-languages.png" alt="Một vài little languages ngẫu nhiên." />

<aside name="little">

Một vài little languages ngẫu nhiên mà bạn có thể bắt gặp.

</aside>

Hầu như mọi dự án phần mềm lớn đều cần một vài ngôn ngữ như vậy. Khi có thể, tốt nhất là tái sử dụng một ngôn ngữ sẵn có thay vì tự tạo. Khi tính cả tài liệu, debugger, hỗ trợ editor, tô sáng cú pháp và các yếu tố khác, thì tự làm sẽ là một nhiệm vụ nặng nề.

Nhưng vẫn có khả năng bạn sẽ cần tự viết parser hoặc công cụ khác khi không có thư viện sẵn có phù hợp. Ngay cả khi bạn tái sử dụng một bản hiện thực có sẵn, sớm muộn gì bạn cũng sẽ phải debug, bảo trì và “mò” vào bên trong nó.

### Ngôn ngữ là bài tập tuyệt vời

Các vận động viên chạy đường dài đôi khi tập luyện với tạ buộc ở mắt cá hoặc ở độ cao nơi không khí loãng. Khi bỏ tạ hoặc xuống vùng không khí giàu oxy, họ sẽ thấy đôi chân nhẹ hơn và chạy xa, chạy nhanh hơn.

Hiện thực hóa một ngôn ngữ là một bài kiểm tra thực sự về kỹ năng lập trình. Code phức tạp và đòi hỏi hiệu năng cao. Bạn phải thành thạo đệ quy, mảng động, cây, đồ thị và hash table. Có thể bạn vẫn dùng hash table hàng ngày, nhưng bạn có *thực sự* hiểu chúng không? Sau khi chúng ta tự tay xây một cái từ đầu, tôi đảm bảo là bạn sẽ hiểu.

Tôi muốn cho bạn thấy rằng một interpreter không đáng sợ như bạn nghĩ, nhưng để hiện thực tốt vẫn là một thử thách. Nếu vượt qua, bạn sẽ trở thành một lập trình viên giỏi hơn, và thông minh hơn trong cách sử dụng cấu trúc dữ liệu và thuật toán trong công việc hàng ngày.

### Thêm một lý do nữa

Lý do cuối này khó nói ra, vì nó rất gần với trái tim tôi. Từ khi còn nhỏ mới học lập trình, tôi đã cảm thấy có điều gì đó “ma thuật” ở ngôn ngữ. Khi lần đầu gõ từng ký tự để viết chương trình BASIC, tôi không thể hình dung nổi BASIC *tự thân* được tạo ra như thế nào.

Sau này, vẻ mặt vừa kính nể vừa sợ hãi của bạn bè đại học khi nói về lớp học compiler đủ để thuyết phục tôi rằng những người làm ngôn ngữ là một “giống loài” khác — kiểu pháp sư được trao quyền tiếp cận những bí thuật.

Đó là một <span name="image">hình ảnh</span> thú vị, nhưng cũng có mặt tối. *Tôi* không cảm thấy mình là pháp sư, nên tôi nghĩ mình thiếu một phẩm chất bẩm sinh nào đó để gia nhập nhóm đó. Dù tôi đã bị cuốn hút bởi ngôn ngữ từ khi còn vẽ nguệch ngoạc những từ khóa tự chế trong vở, phải mất hàng chục năm tôi mới đủ can đảm để thực sự học chúng. Cái “ma thuật” đó, cảm giác độc quyền đó, đã loại trừ *tôi*.

<aside name="image">

Và những người trong nghề cũng không ngần ngại tô đậm hình ảnh này. Hai trong số những cuốn sách kinh điển về ngôn ngữ lập trình có hình [rồng](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools) và [pháp sư](https://mitpress.mit.edu/sites/default/files/sicp/index.html) trên bìa.

</aside>

Khi cuối cùng tôi bắt đầu tự ráp những interpreter nhỏ của mình, tôi nhanh chóng nhận ra rằng, tất nhiên, chẳng có ma thuật nào cả. Chỉ là code, và những người làm ngôn ngữ cũng chỉ là con người.

*Có* một vài kỹ thuật bạn ít gặp ngoài lĩnh vực ngôn ngữ lập trình, và một số phần hơi khó. Nhưng chúng không khó hơn những thử thách khác mà bạn đã từng vượt qua. Tôi hy vọng rằng nếu bạn từng cảm thấy e ngại trước lĩnh vực này, và cuốn sách này giúp bạn vượt qua nỗi sợ đó, thì có lẽ tôi sẽ để lại cho bạn một chút can đảm nhiều hơn trước đây.  

Và, ai mà biết được, có thể *bạn* sẽ tạo ra ngôn ngữ vĩ đại tiếp theo. Phải có ai đó làm chứ.  

## Cấu trúc của cuốn sách  

Cuốn sách này được chia thành ba phần. Bạn đang đọc phần đầu tiên. Đây là vài chương để giúp bạn định hướng, làm quen với một số thuật ngữ mà dân “hacker ngôn ngữ” hay dùng, và giới thiệu cho bạn về Lox — ngôn ngữ mà chúng ta sẽ hiện thực.  

Hai phần còn lại, mỗi phần sẽ xây dựng một interpreter Lox hoàn chỉnh. Trong mỗi phần, các chương đều có cấu trúc giống nhau: mỗi chương tập trung vào một tính năng của ngôn ngữ, giải thích các khái niệm đằng sau nó, rồi hướng dẫn bạn từng bước hiện thực.  

Tôi đã phải thử nghiệm và điều chỉnh khá nhiều, nhưng cuối cùng cũng chia được hai interpreter thành những “miếng” vừa tầm một chương, mỗi chương dựa trên kiến thức của các chương trước nhưng không phụ thuộc vào chương sau. Ngay từ chương đầu tiên, bạn sẽ có một chương trình chạy được để thử nghiệm. Mỗi chương tiếp theo sẽ bổ sung thêm tính năng, cho đến khi bạn có một ngôn ngữ hoàn chỉnh.  

Ngoài phần văn xuôi tiếng Anh dày đặc nhưng (hy vọng là) cuốn hút, các chương còn có một vài điểm thú vị khác:  

### Code

Chúng ta đang nói về việc *crafting* interpreter, nên cuốn sách này chứa code thật. Mỗi một dòng code cần thiết đều được đưa vào, và mỗi đoạn trích (snippet) sẽ cho bạn biết chèn nó vào đâu trong bản hiện thực đang ngày càng lớn của bạn.

Nhiều sách và bản hiện thực ngôn ngữ khác sử dụng các công cụ như [Lex](https://en.wikipedia.org/wiki/Lex_(software)) và <span name="yacc">[Yacc](https://en.wikipedia.org/wiki/Yacc)</span> — những **compiler-compiler** — để tự động sinh ra một số file mã nguồn từ một mô tả cấp cao hơn. Các công cụ này có ưu và nhược điểm, và cũng có những ý kiến mạnh mẽ — thậm chí mang tính “tín ngưỡng” — ở cả hai phía.

<aside name="yacc">

Yacc là một công cụ nhận vào một file grammar và sinh ra file mã nguồn cho compiler, nên nó giống như một “compiler” tạo ra compiler, và từ đó có thuật ngữ “compiler-compiler”.

Yacc không phải là công cụ đầu tiên thuộc loại này, đó là lý do nó được đặt tên là “Yacc” — *Yet Another* Compiler-Compiler. Một công cụ tương tự ra đời sau là [Bison](https://en.wikipedia.org/wiki/GNU_bison), đặt tên chơi chữ dựa trên cách phát âm của Yacc giống “yak”.

<img src="image/introduction/yak.png" alt="Một con yak." />

Nếu bạn thấy những trò tự tham chiếu và chơi chữ này thú vị, bạn sẽ hợp gu ở đây. Nếu không, thì… có lẽ khiếu hài hước của dân mê ngôn ngữ là một “khẩu vị” cần tập quen.

</aside>

Chúng ta sẽ không dùng chúng ở đây. Tôi muốn đảm bảo không có góc tối nào để “ma thuật” hay sự mơ hồ ẩn nấp, nên ta sẽ viết mọi thứ bằng tay. Như bạn sẽ thấy, nó không tệ như nghe có vẻ, và điều đó giúp bạn thực sự hiểu từng dòng code và cách cả hai interpreter hoạt động.

Một cuốn sách có những ràng buộc khác với “thế giới thực”, nên phong cách code ở đây có thể không phải lúc nào cũng phản ánh cách tốt nhất để viết phần mềm sản xuất dễ bảo trì. Nếu tôi có vẻ hơi “tùy tiện” khi bỏ qua `private` hoặc khai báo biến toàn cục, hãy hiểu rằng tôi làm vậy để code dễ đọc hơn. Trang sách không rộng như IDE của bạn và từng ký tự đều đáng giá.

Ngoài ra, code không có nhiều comment. Đó là vì mỗi nhóm dòng code đều được bao quanh bởi vài đoạn văn giải thích chi tiết. Khi bạn viết một cuốn sách đi kèm chương trình của mình, bạn cũng có thể bỏ bớt comment. Còn nếu không, bạn nên dùng `//` nhiều hơn tôi.

Dù cuốn sách chứa mọi dòng code và giải thích ý nghĩa của chúng, nó không mô tả cách biên dịch và chạy interpreter. Tôi giả định bạn có thể tự tạo makefile hoặc project trong IDE ưa thích để chạy code. Những hướng dẫn kiểu đó nhanh lỗi thời, và tôi muốn cuốn sách này “lão hóa” như rượu XO, chứ không như rượu tự nấu ngoài sân.

### Snippet

Vì cuốn sách này chứa *tất cả* các dòng code cần thiết cho bản hiện thực, các snippet được trình bày rất chính xác. Ngoài ra, vì tôi cố giữ chương trình ở trạng thái chạy được ngay cả khi thiếu các tính năng lớn, đôi khi ta sẽ thêm code tạm thời rồi thay thế nó ở các snippet sau.

Một snippet “đầy đủ phụ kiện” trông như thế này:

<div class="codehilite"><pre class="insert-before">
      default:
</pre><div class="source-file"><em>lox/Scanner.java</em><br>
in <em>scanToken</em>()<br>
replace 1 line</div>
<pre class="insert">
        <span class="k">if</span> (<span class="i">isDigit</span>(<span class="i">c</span>)) {
          <span class="i">number</span>();
        } <span class="k">else</span> {
          <span class="t">Lox</span>.<span class="i">error</span>(<span class="i">line</span>, <span class="s">&quot;Unexpected character.&quot;</span>);
        }
</pre><pre class="insert-after">
        break;
</pre></div>
<div class="source-file-narrow"><em>lox/Scanner.java</em>, in <em>scanToken</em>(), replace 1 line</div>

Ở giữa là đoạn code mới cần thêm. Có thể sẽ có vài dòng mờ phía trên hoặc dưới để cho bạn thấy nó nằm ở đâu trong code hiện có. Ngoài ra còn có một chú thích nhỏ cho biết file nào và vị trí chèn snippet. Nếu chú thích ghi “replace _ lines”, nghĩa là có một đoạn code hiện tại giữa các dòng mờ cần xóa và thay bằng snippet mới.

### Aside

Các <span name="joke">aside</span> chứa tiểu sử, bối cảnh lịch sử, tham chiếu đến các chủ đề liên quan và gợi ý những hướng khám phá khác. Không có gì trong đó là *bắt buộc* để hiểu các phần sau của sách, nên nếu muốn bạn có thể bỏ qua. Tôi sẽ không phán xét, nhưng có thể sẽ hơi buồn.

<aside name="joke">

Thực ra, một số aside có nội dung hữu ích. Nhưng phần lớn chỉ là mấy trò đùa ngớ ngẩn và hình vẽ nghiệp dư.

</aside>

### Challenge

Mỗi chương kết thúc bằng một vài bài tập. Khác với các bài tập ôn tập trong sách giáo khoa, những bài này giúp bạn học *nhiều hơn* những gì có trong chương. Chúng buộc bạn rời khỏi “lộ trình có hướng dẫn” và tự mình khám phá. Chúng sẽ khiến bạn phải nghiên cứu các ngôn ngữ khác, tìm cách hiện thực tính năng, hoặc đơn giản là bước ra khỏi vùng an toàn.

<span name="warning">Chinh phục</span> các thử thách này và bạn sẽ có thêm hiểu biết rộng hơn, cùng vài “vết xước” nho nhỏ. Hoặc bỏ qua nếu bạn muốn ở lại trong sự thoải mái của “xe buýt du lịch”. Đây là cuốn sách của bạn.

<aside name="warning">

Lưu ý: các challenge thường yêu cầu bạn thay đổi interpreter đang xây dựng. Bạn nên thực hiện chúng trên một bản sao code của mình. Các chương sau giả định interpreter của bạn vẫn ở trạng thái nguyên bản (“chưa bị thử thách”).

</aside>

### Design note

Hầu hết sách về “ngôn ngữ lập trình” đều chỉ là sách về *hiện thực* ngôn ngữ. Họ hiếm khi bàn về cách *thiết kế* ngôn ngữ đó. Việc hiện thực thú vị vì nó được <span name="benchmark">định nghĩa chính xác</span>. Lập trình viên chúng ta thường có xu hướng thích những thứ trắng đen rõ ràng, 1 và 0.

<aside name="benchmark">

Tôi biết nhiều “hacker ngôn ngữ” sống bằng nghề này. Bạn trượt một bản đặc tả ngôn ngữ qua khe cửa họ, chờ vài tháng, và nhận lại code cùng kết quả benchmark.

</aside>

Cá nhân tôi nghĩ thế giới chỉ cần bấy nhiêu bản hiện thực của <span name="fortran">FORTRAN 77</span> là đủ. Đến một lúc nào đó, bạn sẽ thấy mình đang thiết kế một ngôn ngữ *mới*. Khi chơi “trò” đó, yếu tố con người trở nên quan trọng: tính năng nào dễ học, làm sao cân bằng giữa đổi mới và quen thuộc, cú pháp nào dễ đọc hơn và với ai.

<aside name="fortran">

Hy vọng ngôn ngữ mới của bạn không “cứng mã” giả định về chiều rộng của thẻ đục lỗ vào grammar.

</aside>

Tất cả những yếu tố đó ảnh hưởng sâu sắc đến thành công của ngôn ngữ mới. Tôi muốn ngôn ngữ của bạn thành công, nên ở một số chương tôi sẽ kết thúc bằng “design note” — một tiểu luận nhỏ về một góc độ con người trong ngôn ngữ lập trình. Tôi không phải chuyên gia — mà tôi cũng không chắc có ai thực sự là chuyên gia — nên hãy đón nhận chúng với một chút hoài nghi. Điều đó sẽ khiến chúng trở thành món ăn tinh thần “đậm đà” hơn, và đó là mục tiêu chính của tôi.

## Interpreter đầu tiên

Chúng ta sẽ viết interpreter đầu tiên, **jlox**, bằng <span name="lang">Java</span>. Trọng tâm ở đây là *các khái niệm*. Ta sẽ viết code đơn giản và rõ ràng nhất có thể để hiện thực chính xác ngữ nghĩa của ngôn ngữ. Điều này sẽ giúp ta làm quen với các kỹ thuật cơ bản và mài giũa sự hiểu biết về cách ngôn ngữ được kỳ vọng hoạt động.

<aside name="lang">

Cuốn sách này dùng Java và C, nhưng đã có độc giả port code sang [nhiều ngôn ngữ khác](https://github.com/munificent/craftinginterpreters/wiki/Lox-implementations). Nếu bạn không thích các ngôn ngữ tôi chọn, hãy thử xem qua chúng.

</aside>

Java là một ngôn ngữ tuyệt vời cho mục đích này. Nó đủ cấp cao để ta không bị ngợp bởi các chi tiết hiện thực vụn vặt, nhưng vẫn đủ tường minh. Không giống các ngôn ngữ scripting, thường có nhiều cơ chế phức tạp ẩn bên dưới, Java cho ta kiểu tĩnh để thấy rõ mình đang làm việc với cấu trúc dữ liệu nào.

Tôi cũng chọn Java vì nó là ngôn ngữ hướng đối tượng. Mô hình này đã quét qua thế giới lập trình vào thập niên 90 và giờ là cách tư duy chủ đạo của hàng triệu lập trình viên. Nhiều khả năng bạn đã quen với việc tổ chức code thành class và method, nên ta sẽ giữ bạn trong vùng an toàn đó.

Dù giới học thuật về ngôn ngữ đôi khi xem nhẹ ngôn ngữ hướng đối tượng, thực tế là chúng vẫn được dùng rộng rãi ngay cả trong lĩnh vực này. GCC và LLVM được viết bằng C++, cũng như hầu hết các máy ảo JavaScript. Ngôn ngữ hướng đối tượng ở khắp nơi, và công cụ, compiler *cho* một ngôn ngữ thường được viết *bằng chính* <span name="host">ngôn ngữ đó</span>.

<aside name="host">

Compiler đọc file ở một ngôn ngữ, dịch chúng và xuất ra file ở ngôn ngữ khác. Bạn có thể hiện thực compiler bằng bất kỳ ngôn ngữ nào, kể cả chính ngôn ngữ mà nó biên dịch — quá trình này gọi là **self-hosting**.

Bạn chưa thể biên dịch compiler của mình bằng chính nó, nhưng nếu bạn có một compiler khác cho ngôn ngữ đó được viết bằng ngôn ngữ khác, bạn có thể dùng *nó* để biên dịch compiler của mình một lần. Giờ bạn có thể dùng bản đã biên dịch của compiler của mình để biên dịch các phiên bản tương lai của chính nó, và bỏ bản gốc được biên dịch từ compiler khác. Quá trình này gọi là **bootstrapping**, xuất phát từ hình ảnh “tự kéo mình lên bằng quai ủng của chính mình”.

<img src="image/introduction/bootstrap.png" alt="Sự thật: Đây là phương tiện di chuyển chính của cao bồi Mỹ." />

</aside>

Và cuối cùng, Java cực kỳ phổ biến. Điều đó nghĩa là nhiều khả năng bạn đã biết nó, nên sẽ ít thứ mới phải học để bắt đầu. Nếu bạn chưa quen Java, đừng lo. Tôi cố gắng chỉ dùng một tập con tối giản. Tôi dùng toán tử diamond từ Java 7 để code gọn hơn, và đó gần như là tất cả các tính năng “nâng cao” tôi dùng. Nếu bạn biết một ngôn ngữ hướng đối tượng khác như C# hoặc C++, bạn vẫn có thể theo kịp.

Kết thúc Phần II, ta sẽ có một bản hiện thực đơn giản, dễ đọc. Nó không nhanh lắm, nhưng đúng. Tuy nhiên, ta chỉ làm được điều đó nhờ tận dụng các cơ chế runtime sẵn có của Java VM. Ta muốn học cách chính Java *tự* hiện thực những thứ đó.

## Interpreter thứ hai

Ở phần tiếp theo, ta sẽ bắt đầu lại từ đầu, nhưng lần này là với C. C là ngôn ngữ hoàn hảo để hiểu cách một bản hiện thực *thực sự* hoạt động, từ byte trong bộ nhớ đến dòng lệnh chạy qua CPU.

Một lý do lớn để dùng C là tôi muốn cho bạn thấy những gì C làm đặc biệt tốt, nhưng điều đó *cũng* nghĩa là bạn cần khá thoải mái với nó. Bạn không cần phải là “hóa thân” của Dennis Ritchie, nhưng cũng không nên sợ pointer.

Nếu bạn chưa sẵn sàng, hãy đọc một cuốn nhập môn C và luyện qua, rồi quay lại đây. Đổi lại, bạn sẽ rời cuốn sách này với kỹ năng C tốt hơn. Điều này hữu ích vì rất nhiều bản hiện thực ngôn ngữ được viết bằng C: Lua, CPython, MRI của Ruby, v.v.

Trong interpreter C của chúng ta, <span name="clox">clox</span>, ta buộc phải tự hiện thực mọi thứ mà Java cho sẵn. Ta sẽ viết mảng động và hash table của riêng mình. Ta sẽ quyết định cách object được biểu diễn trong bộ nhớ, và xây dựng một garbage collector để thu hồi chúng.

<aside name="clox">

Tôi phát âm tên này như “si-locks”, nhưng bạn có thể đọc là “clocks” hoặc thậm chí “cloch” (phát âm “x” như tiếng Hy Lạp) nếu bạn thích.

</aside>

Bản Java tập trung vào tính đúng đắn. Giờ khi đã có điều đó, ta sẽ hướng tới *tốc độ*. Interpreter C sẽ có một <span name="compiler">compiler</span> dịch Lox sang bytecode hiệu quả (đừng lo, tôi sẽ giải thích sớm thôi), rồi execute nó. Đây là kỹ thuật được dùng bởi Lua, Python, Ruby, PHP và nhiều ngôn ngữ thành công khác.

<aside name="compiler">

Bạn nghĩ đây chỉ là sách về interpreter thôi sao? Nó còn là sách về compiler nữa. Hai trong một!

</aside>

Ta sẽ thử cả benchmark và tối ưu hóa. Kết thúc, ta sẽ có một interpreter mạnh mẽ, chính xác, nhanh cho ngôn ngữ của mình, đủ sức sánh với các bản hiện thực chuyên nghiệp khác. Không tệ cho một cuốn sách và vài nghìn dòng code.

<div class="challenges">

## Thử thách

1.  Có ít nhất sáu domain-specific language được dùng trong [hệ thống nhỏ tôi ráp lại](https://github.com/munificent/craftinginterpreters) để viết và xuất bản cuốn sách này. Chúng là gì?

2.  Viết và chạy chương trình “Hello, world!” bằng Java. Thiết lập makefile hoặc project trong IDE để chạy được. Nếu có debugger, hãy làm quen với nó và thử bước qua chương trình khi chạy.

3.  Làm tương tự với C. Để luyện pointer, hãy định nghĩa một [doubly linked list](https://en.wikipedia.org/wiki/Doubly_linked_list) chứa các string được cấp phát trên heap. Viết hàm chèn, tìm và xóa phần tử. Kiểm tra chúng.

</div>

<div class="design-note">

## Ghi chú thiết kế: Tên gọi có gì?

Một trong những thử thách khó nhất khi viết cuốn sách này là nghĩ tên cho ngôn ngữ mà nó hiện thực. Tôi đã duyệt *hàng trang* ứng viên trước khi tìm được một cái phù hợp. Như bạn sẽ thấy ngay ngày đầu bắt tay vào xây ngôn ngữ của riêng mình, đặt tên là một việc khó nhằn. Một cái tên tốt cần thỏa vài tiêu chí:

1.  **Chưa bị dùng.** Bạn có thể gặp đủ loại rắc rối pháp lý và xã hội nếu vô tình “dẫm” lên tên của ai đó.

2.  **Dễ phát âm.** Nếu mọi thứ suôn sẻ, sẽ có rất nhiều người nói và viết tên ngôn ngữ của bạn. Bất cứ thứ gì dài hơn vài âm tiết hoặc vài chữ cái sẽ khiến họ khó chịu.

3.  **Dễ tìm kiếm.** Người ta sẽ Google tên ngôn ngữ của bạn để tìm hiểu, nên bạn muốn một từ đủ hiếm để hầu hết kết quả trỏ về tài liệu của bạn. Dù với lượng AI mà các công cụ tìm kiếm đang tích hợp, điều này ít quan trọng hơn, nhưng bạn vẫn không giúp gì cho người dùng nếu đặt tên ngôn ngữ là “for”.

4.  **Không mang nghĩa tiêu cực ở nhiều nền văn hóa.** Khó để kiểm soát hết, nhưng đáng cân nhắc. Nhà thiết kế Nimrod đã đổi tên ngôn ngữ thành “Nim” vì quá nhiều người nhớ rằng Bugs Bunny dùng “Nimrod” như một lời xúc phạm (dù Bugs dùng theo nghĩa mỉa mai).

Nếu cái tên tiềm năng vượt qua được “ải” này, hãy giữ nó. Đừng quá sa đà vào việc tìm một cái tên thể hiện trọn vẹn “bản chất” ngôn ngữ của bạn. Nếu tên của các ngôn ngữ thành công khác dạy ta điều gì, thì đó là tên không quan trọng lắm. Bạn chỉ cần một “token” đủ độc đáo là được.

</div>