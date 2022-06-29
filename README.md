# API HOOKING	

## I. API HOOKING LÀ GÌ?

_API Hooking là một kỹ thuật mà sử dụng lệnh **JMP** hoặc **CALL** để thay đổi luồng chạy của các API. API Hooking có thể thực hiện bằng nhiều phương pháp, trong bài viết này thì chúng tôi sẽ nghiên cứu cụ thể với hệ điều hành Windows trên nền tảng 32 bit._


## II. Ý TƯỞNG	

Xét một chương trình đơn giản như sau:

```C++
#include <iostream>

int add_two_ints(int a, int b) {
	return a + b;
}

int main() {
	std::cout << add(1, 2) << std::endl;
	return 0;
}
```

Chúng ta có một hàm **add_two_ints** thực hiện tính tổng 2 tham số a, b được truyền vào. Và ở hàm **main** thực hiện call **add_two_ints** với tham số `a = 1` và `b = 2`. Sau đó in ra kết quả.

Giờ tôi sẽ chuyển nó qua mã assembly và phân tích.

```x86asm
add_two_ints proc near
push    ebp
mov     ebp, esp
mov     eax, [ebp+8] ; a
add     eax, [ebp+C] ; b
pop     ebp
retn
add_two_ints endp

_main proc near
push    ebp
mov     ebp, esp
push    offset std_end
push    2               ; b
push    1               ; a
call    add_two_ints
add     esp, 8
push    eax
mov     ecx, ds:"std::ostream std::cout"
call    ds:"std::ostream::operator<<(int)"
mov     ecx, eax
call    ds:"std::ostream::operator<<(std::ostream & (*)(std::ostream &))"
xor     eax, eax
pop     ebp
retn
_main endp
```

0. Ban đầu hãy coi stack của chúng ta đang trống rỗng | `stack = []`.

1. Bỏ qua phần header của hàm main, thì chúng ta gặp câu lệnh đầu tiên là `push offset std_end`. Lúc này object std_end sẽ được push vào stack. | `stack = [std_end]`.

2. Tương tự với 2 câu lệnh `push` phía sau, chính là push 2 tham số cho hàm `add_two_ints`. | `stack = [std_end, 2, 1]`.

3. Sau đó `call add_two_ints` được chạy để push thêm `return address` và jump đến hàm mục tiêu. | `stack = [std_end, 2, 1, main + 11 (add esp, 8)]`.

4. Đi vào hàm `add_two_ints` chúng ta gặp function header (chính là 2 dòng `push ebp` và `mov ebp, esp`) | `stack = [std_end, 2, 1, main + 11 (add esp, 8), old_ebp]`

5. Hai dòng tiếp theo thực hiện đưa giá trị của a vào eax, sau đó cộng giá trị của eax với b. Mà eax cũng là thanh ghi thường được dùng để trả về kết quả.

6. Hai dòng cuối cùng là function footer (dùng để backup lại giá trị của ebp) và câu lệnh ret dùng để nhảy về vị trí sau chỗ gọi hàm (cụ thể là giá trị tại top của stack, sau đó giá trị này bị pop). | `stack = [std_end, 2, 1]`

7. Sau khi return thì chúng ta thấy có một dòng `add esp, 8`, dòng này để dọn dẹp stack sau khi gọi hàm. esp là thanh ghi chỉ vào top của stack, add esp lên 8 đơn vị sẽ bỏ qua 8 bytes ở đầu stack và stack lúc này chỉ còn `stack = [std_end]`.

8. Những dòng sau chỉ dùng để in giá trị của eax, nên chúng ta sẽ bỏ qua...

Vậy là chúng ta đã cùng phân tích luồng chạy của chương trình này. Tiếp theo sẽ là ý tưởng thực hiện hooking, nếu giả sử hàm này là một hàm trong game đối kháng, hàm này thực hiện tính tổng của sát thương cơ bản với sát thương thêm. Sau đó trả về tổng 2 tham số này để áp dụng lượng sát thương này lên kẻ địch. Mình sẽ khai thác nó để làm cho hàm này trả về một giá trị cao hơn.

## III. TẤN CÔNG

Ý tưởng thực hiện tấn công ở đây là thay vì để hàm chạy thẳng một mạch như ban đầu, thì lúc thực hiện hàm mình sẽ bắt chương trình phải nhảy sang đoạn mã khác mà mình ghi vào ở một nơi nào đó trong bộ nhớ, ở hàm đó sẽ thực hiện trả về giá trị là a * b thay vì a + b.

Quá trình làm cụ thể sẽ như sau:

1. Tạo một vùng nhớ và để ghi code `custom_function`. Nếu là Windows thì vùng nhớ này cần có quyền `EXECUTE-READ-WRITE`.

![](1.png)

2. Sau đó tại đầu hàm `add_two_ints` ghi đè lên đó một mã `JMP` sang `custom_function`. Hàm này sẽ tính `a * b` sau đó trả về kết quả và quay về hàm `main`.

![](2.png)

## IV. CÁC LOẠI API HOOKING KHÁC

1. Hook Mid Function
   
   Với ví dụ bên trên là một ví dụ đơn giản để giới thiệu về API Hooking, thực tế thì khi hook chúng ta không hoàn toàn thay thế hàm cũ, mà thay vào thì sau khi `custom_function` chạy xong thì sẽ jump back về vị trí nối tiếp của hàm cũ.
	
2. Detours
   
   Detours là một phương pháp phổ biến và tiện lợi hơn những phương pháp hooking khác. Với những phương pháp đã kể trên thì khi bạn muốn hook một hàm `custom_function` vào một hàm `target_function` nào đó. Thì code của custom_function của bạn đôi khi phải lồng thêm mã `asm` khiến code trở lên phức tạp hơn. Thay vào đó, với phương pháp detours thì bạn chỉ việc code như bình thường bằng ngôn ngữ bậc cao.











