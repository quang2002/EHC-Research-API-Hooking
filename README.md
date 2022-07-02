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

	Ví dụ với một game bắn súng không ai biết đến đó là AssaultCube. Tôi tìm thấy một vị trí mà tại đó mỗi khi mình bắn súng thì đạn sẽ bị giảm đi.

	```x86asm
	ac_client.exe+C726D - mov eax,[ac_client.exe+17F0DC]
	ac_client.exe+C7272 - cmp ecx,eax
	ac_client.exe+C7274 - cmovl eax,ecx
	ac_client.exe+C7277 - mov ecx,edi
	ac_client.exe+C7279 - sub ebx,eax
	ac_client.exe+C727B - mov edx,ebx
	ac_client.exe+C727D - call ac_client.exe+CB770
	ac_client.exe+C7282 - mov eax,[esi+14]
	ac_client.exe+C7285 - mov ecx,[esi+08]
	ac_client.exe+C7288 - cmp dword ptr [eax],00
	ac_client.exe+C728B - jne ac_client.exe+C7305
	ac_client.exe+C728D - test ecx,ecx
	ac_client.exe+C728F - je ac_client.exe+C72CD
	ac_client.exe+C7291 - xor eax,eax
	ac_client.exe+C7293 - mov [esp+2C],ecx
	ac_client.exe+C7297 - cmp ecx,[ac_client.exe+18AC00]
	ac_client.exe+C729D - push 00
	ac_client.exe+C729F - push ecx
	ac_client.exe+C72A0 - sete al
	ac_client.exe+C72A3 - mov [esp],00000000
	ac_client.exe+C72AA - inc eax
	ac_client.exe+C72AB - mov [esp+30],00000001
	ac_client.exe+C72B3 - push eax
	ac_client.exe+C72B4 - lea eax,[esp+30]
	ac_client.exe+C72B8 - mov [esp+30],ac_client.exe+15B00C
	ac_client.exe+C72C0 - push eax
	ac_client.exe+C72C1 - push 18
	ac_client.exe+C72C3 - mov ecx,ac_client.exe+17C130
	ac_client.exe+C72C8 - call ac_client.exe+116A50
	ac_client.exe+C72CD - mov eax,[esi+18]
	ac_client.exe+C72D0 - add [eax],000000FA
	ac_client.exe+C72D6 - mov eax,[esi+08]
	ac_client.exe+C72D9 - mov [eax+00000374],00000000
	ac_client.exe+C72E3 - mov [esi+1C],00000000
	ac_client.exe+C72EA - cmp dword ptr [ac_client.exe+190908],00
	ac_client.exe+C72F1 - je ac_client.exe+C7262
	ac_client.exe+C72F7 - mov eax,[esi+08]
	ac_client.exe+C72FA - cmp eax,[ac_client.exe+18AC00]
	ac_client.exe+C7300 - jmp ac_client.exe+C724F
	ac_client.exe+C7305 - mov [ecx+00000374],esi
	ac_client.exe+C730B - mov eax,[esi+0C]
	ac_client.exe+C730E - inc [esi+1C]
	ac_client.exe+C7311 - cmp byte ptr [eax+66],00
	ac_client.exe+C7315 - jne ac_client.exe+C7321
	ac_client.exe+C7317 - mov eax,[esi+08]
	ac_client.exe+C731A - mov byte ptr [eax+00000204],00
	ac_client.exe+C7321 - mov eax,[esi+04]
	ac_client.exe+C7324 - mov eax,[eax*4+ac_client.exe+1837F0]
	ac_client.exe+C732B - test eax,eax
	ac_client.exe+C732D - jle ac_client.exe+C733E
	ac_client.exe+C732F - cmp [esi+1C],eax
	ac_client.exe+C7332 - jl ac_client.exe+C733E
	ac_client.exe+C7334 - mov eax,[esi+08]
	ac_client.exe+C7337 - mov byte ptr [eax+00000204],00
	ac_client.exe+C733E - mov eax,[esi+08]
	ac_client.exe+C7341 - lea ecx,[esp+18]
	ac_client.exe+C7345 - push ecx
	ac_client.exe+C7346 - lea ecx,[esp+10]
	ac_client.exe+C734A - push ecx
	ac_client.exe+C734B - movq xmm0,[eax+04]
	ac_client.exe+C7350 - mov ecx,esi
	ac_client.exe+C7352 - movq [esp+14],xmm0
	ac_client.exe+C7358 - mov eax,[eax+0C]
	ac_client.exe+C735B - mov [esp+1C],eax
	ac_client.exe+C735F - mov eax,[esp+3C]
	ac_client.exe+C7363 - movq xmm0,[eax]
	ac_client.exe+C7367 - mov eax,[eax+08]
	ac_client.exe+C736A - movq [esp+20],xmm0
	ac_client.exe+C7370 - movss xmm0,[esp+1C]
	ac_client.exe+C7376 - subss xmm0,[ac_client.exe+15BB88]
	ac_client.exe+C737E - mov [esp+28],eax
	ac_client.exe+C7382 - mov eax,[esi]
	ac_client.exe+C7384 - movss [esp+1C],xmm0
	ac_client.exe+C738A - call dword ptr [eax+14]
	ac_client.exe+C738D - mov eax,[esi+08]
	ac_client.exe+C7390 - cmp eax,[ac_client.exe+18AC00]
	ac_client.exe+C7396 - jne ac_client.exe+C73AF
	ac_client.exe+C7398 - push [esi+04]
	ac_client.exe+C739B - push ac_client.exe+12E994
	ac_client.exe+C73A0 - push ac_client.exe+1550C4
	ac_client.exe+C73A5 - push 02
	ac_client.exe+C73A7 - call ac_client.exe+D44C0
	ac_client.exe+C73AC - add esp,10
	ac_client.exe+C73AF - mov [ac_client.exe+19091C],00000000
	ac_client.exe+C73B9 - lea edx,[esp+18]
	ac_client.exe+C73BD - push [esi+08]
	ac_client.exe+C73C0 - lea ecx,[esp+10]
	ac_client.exe+C73C4 - call ac_client.exe+C9510
	ac_client.exe+C73C9 - mov eax,[esi]
	ac_client.exe+C73CB - lea ecx,[esp+1C]
	ac_client.exe+C73CF - add esp,04
	ac_client.exe+C73D2 - push 00
	ac_client.exe+C73D4 - push ecx
	ac_client.exe+C73D5 - lea ecx,[esp+14]
	ac_client.exe+C73D9 - push ecx
	ac_client.exe+C73DA - mov ecx,esi
	ac_client.exe+C73DC - call dword ptr [eax+10]
	ac_client.exe+C73DF - mov eax,[esi+0C]
	ac_client.exe+C73E2 - push ebx
	ac_client.exe+C73E3 - movsx ecx,word ptr [eax+48]
	ac_client.exe+C73E7 - mov eax,[esi+18]
	ac_client.exe+C73EA - mov [eax],ecx
	ac_client.exe+C73EC - mov eax,[esi+14]
	ac_client.exe+C73EF - dec [eax]					<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< HERE
	ac_client.exe+C73F1 - lea eax,[esp+1C]
	ac_client.exe+C73F5 - push eax					<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< RETURN ADDRESS
	ac_client.exe+C73F6 - push ecx
	ac_client.exe+C73F7 - mov ecx,esi
	ac_client.exe+C73F9 - call ac_client.exe+C9140
	ac_client.exe+C73FE - pop edi
	ac_client.exe+C73FF - pop esi
	ac_client.exe+C7400 - mov al,01
	ac_client.exe+C7402 - pop ebx
	ac_client.exe+C7403 - add esp,24
	ac_client.exe+C7406 - ret 0004
	```
	Chúng ta có thể thấy tại `ac_client.exe+C73EF` thực hiện giảm giá trị của `[eax]`, ở đấy `eax` chính là địa chỉ của vùng nhớ lưu số lượng đạn của chúng ta hiện tại. 

	Ý tưởng của tôi ở đây sẽ đặt một hook tại vị trí này để tăng giá trị của `[eax]` lên mỗi khi đạn bắn. Trong ví dụ này tôi sẽ sử dụng code `C++` để mô tả thao tác:
	
	Bước 1: Tạo một vùng nhớ để lưu đoạn mã tăng `[eax]`
	
	```C++
	// Cấp phát một vùng nhớ 1024 bytes, có quyền execute-read-write
	void * buffer = VirtualAlloc(NULL, 1024, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

	// Ghi đoạn mã tăng eax lên một đơn vị "add [eax]", câu lệnh này chỉ mất 2 bytes và 2 bytes đó là FF 00
	// Mà một lệnh jmp cần 5 bytes, vậy nên sẽ làm mất câu lệnh "lea eax,[esp+1C]" ngay sau dec [eax]. Nên tôi sẽ đem cả 4 bytes của câu lệnh này qua buffer để hook có thể chạy mượt mà.

	unsigned char opcodes[] = {
		0xFF, 0x00,					// inc [eax]
		0x8D, 0x44, 0x24, 0x1C		// lea eax,[esp+1C]
	};
	
	memcpy(buffer, opcodes, sizeof(opcodes));

	// Sau khi có đủ đoạn mã để cheat rồi thì chúng ta cần quay về "RETURN ADDRESS" ac_client.exe+C73F5
	unsigned long return_address = ac_client + 0xC73F5;

	// Một lệnh JMP sẽ có format là E9 ?? ?? ?? ?? (4 bytes sau là relative-address)
	*(unsigned char*)(buffer + 0x6) = 0xE9;

	// Và 4 bytes sau là relative address có công thức tính là destination - source - 5
	*(unsigned long*)(buffer + 0x7) = return_address - (unsigned long)(buffer + 0x6) - 5;

	// Lúc này những gì chúng ta tạo ra sẽ trông như sau:

	/*
		...
		ac_client.exe+C73E7 - mov eax,[esi+18]
		ac_client.exe+C73EA - mov [eax],ecx
		ac_client.exe+C73EC - mov eax,[esi+14]
		ac_client.exe+C73EF - dec [eax]					<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< HERE
		ac_client.exe+C73F1 - lea eax,[esp+1C]
		ac_client.exe+C73F5 - push eax					<<<<<<<<<<<<<<<<<<<#
		ac_client.exe+C73F6 - push ecx									   #
		ac_client.exe+C73F7 - mov ecx,esi								   #
		ac_client.exe+C73F9 - call ac_client.exe+C9140					   #
		ac_client.exe+C73FE - pop edi									   #
		ac_client.exe+C73FF - pop esi									   #
		...																   #
																		   #
		buffer + 0			- inc [eax]									   #
		buffer + 2			- lea eax,[esp+1C]							   #
		buffer + 6			- jmp ac_client.exe+C73F5	>>>>>>>>>>>>>>>>>>>#
	*/
	```

	Bước 2: Jmp từ vị trí cần hook qua buffer vừa tạo
	
	```C++
	// vị trí cần hook có address là ac_client.exe+C73EF
	void * hook_address = (void *)(ac_client + 0xC73EF);

	// cấp quyền được ghi vào vùng nhớ này
	unsigned long old_protect;
	VirtualProtect(hook_address, 1024, PAGE_EXECUTE_READWRITE, &old_protect);

	// thực hiện jmp đến buffer
	*(unsigned char*)(hook_address + 0) = 0xE9;
	*(unsigned long*)(hook_address + 1) = (unsigned long)buffer - (unsigned long)hook_address - 5;

	// trả lại quyền ban đầu cho vùng nhớ này
	VirtualProtect(hook_address, 1024, old_protect, NULL);

	// Tổng thể lại kết quả sẽ trông như thế này:

	/*
		...
		ac_client.exe+C73E7 - mov eax,[esi+18]
		ac_client.exe+C73EA - mov [eax],ecx
		ac_client.exe+C73EC - mov eax,[esi+14]
	#<< ac_client.exe+C73EF - dec [eax]
	#   ac_client.exe+C73F1 - lea eax,[esp+1C]
	#   ac_client.exe+C73F5 - push eax					<<<<<<<<<<<<<<<<<<<#
	#   ac_client.exe+C73F6 - push ecx									   #
	#   ac_client.exe+C73F7 - mov ecx,esi								   #
	#   ac_client.exe+C73F9 - call ac_client.exe+C9140					   #
	#   ac_client.exe+C73FE - pop edi									   #
	#   ac_client.exe+C73FF - pop esi									   #
	#   ...																   #
	#   															       #
	#>> buffer + 0			- inc [eax]									   #
	    buffer + 2			- lea eax,[esp+1C]							   #
	    buffer + 6			- jmp ac_client.exe+C73F5	>>>>>>>>>>>>>>>>>>>#
	*/
	```

2. Detours
   
   Detours là một phương pháp phổ biến và tiện lợi hơn những phương pháp hooking khác. Với những phương pháp đã kể trên thì khi bạn muốn hook một hàm `custom_function` vào một hàm `target_function` nào đó. Thì code của custom_function của bạn đôi khi phải lồng thêm mã `asm` khiến code trở lên phức tạp hơn. Thay vào đó, với phương pháp detours thì bạn chỉ việc code như bình thường bằng ngôn ngữ bậc cao.




