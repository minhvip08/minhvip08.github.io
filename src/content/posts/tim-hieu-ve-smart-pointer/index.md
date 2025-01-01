---
title: Smart Pointer - Giải Pháp Quản Lý Bộ Nhớ Tối Ưu Trong C++
published: 2024-12-07
description: Smart pointer là một đối tượng thuộc các lớp được thiết kế để quản lý tài nguyên, như vùng nhớ động hoặc các tài nguyên hệ thống khác, theo cách tự động và an toàn. 
image: './img/smart_pointer_illustration.webp'
tags: [C++, learn, pointer, smartpointer]
category: C++
draft: false 
lang: 'vi'
---
#### Prerequisites: [Pointer trong C++](https://blog.simi.ovh/posts/pointer-trong-c++/)

## Mục lục

1. [Dẫn nhập](#dẫn-nhập)  
2. [Smart Pointer - Giải pháp tối ưu để quản lý bộ nhớ](#smart-pointer---giải-pháp-tối-ưu-để-quản-lý-bộ-nhớ)  
3. [Các loại Smart Pointer](#các-loại-smart-pointer)  
   - [std::unique_ptr]()  
   - [std::shared_ptr]()  
   - [std::weak_ptr]()  
4. [Bảng so sánh 3 loại Smart Pointer](#bảng-so-sánh-3-loại-smart-pointer)  
5. [Sử dụng Smart Pointer sao cho đúng cách?](#sử-dụng-smart-pointer-sao-cho-đúng-cách)  
   - [Sử dụng `std::make_shared`](#sử-dụng-stdmakeshared-thay-vì-khởi-tạo-trực-tiếp)  
   - [Tránh vòng tham chiếu với `std::weak_ptr`](#tránh-vòng-tham-chiếu-với-stdweak_ptr)  
   - [Cân nhắc giữa các loại Smart Pointer](#cân-nhắc-giữa-việc-sử-dụng-các-loại-smart-pointer)  
6. [Kết luận](#kết-luận)  
7. [Tài liệu tham khảo](#tài-liệu-tham-khảo)  

### Dẫn nhập

Trong việc lập trình C++, việc quản lý bộ nhớ luôn là thách thức lớn do đây là ngôn ngữ không có Garbage Collector như Java hay Python, đặc biệt đối với sinh viên hoặc lập trình viên khi mới làm quen ngôn ngữ này. Một trong những nguyên nhân chủ yếu dẫn đến các vấn đề nghiêm trọng trong ứng dụng là việc sử dụng con trỏ thô (*raw pointers*) không đúng cách. Dưới đây là một số vấn đề thường gặp:

- **Rò rỉ bộ nhớ** (*memory leaks*): Khi bộ nhớ được cấp phát động bằng `new` hoặc `malloc` mà không được giải phóng đúng cách.
- **Truy cập ngoài phạm vi** (*dangling pointers*): Khi cố gắng sử dụng một con trỏ đã bị xóa hoặc chưa được khởi tạo.

Ngoài ra, mình còn thường xuyên gặp một số lỗi thường xuyên khác như wild pointer (pointer chưa bao giờ khởi tạo để trỏ vào một vùng nhớ)… Hậu quả của việc quản lý không tôt có thể dẫn đến **undefined behavior** hoặc tệ hơn là **Segmentation Fault (Core dump)**, một lỗi siêu khó chịu mà mình thường xuyên phải sử dụng GDB để debug.

### **Smart Pointer - Giải pháp tối ưu để quản lý bộ nhớ**

Smart pointer (được giới thiệu lần đầu `C++11`) là một đối tượng thuộc các lớp được thiết kế để quản lý tài nguyên, như vùng nhớ động hoặc các tài nguyên hệ thống khác, theo cách tự động và an toàn. Thay vì yêu cầu lập trình viên phải quản lý việc cấp phát và giải phóng bộ nhớ, smart pointers đảm nhận nhiệm vụ này thông qua **quy tắc phạm vi (*scope-based resource management*)**. Khi smart pointer ra khỏi phạm vi sử dụng (scope), nó tự động giải phóng tài nguyên mà nó quản lý, giảm thiểu nguy cơ rò rỉ bộ nhớ (*memory leaks*) hoặc lỗi truy cập trái phép (*dangling pointers*).  

Hiểu đơn giản là smart pointer là một **wrapper class** trên cái **raw pointer**, giúp tự động quản lý vùng nhớ thay vì thủ công qua những dòng code new và delete.

Đây cũng là nguyên lý của một kĩ thuật nổi tiếng có tên là RAII (*Resource Acquisition Is Initialization*):

> RAII (Resource Acquisition Is Initialization) là một nguyên tắc thiết kế trong lập trình C++, trong đó việc **quản lý tài nguyên** (như bộ nhớ, file, socket, hoặc khóa mutex) được gắn liền với **vòng đời của một đối tượng**. Tài nguyên sẽ được cấp phát (acquired) khi đối tượng được khởi tạo (**constructor**) và tự động được giải phóng (released) khi đối tượng bị hủy (**destructor**).
> 

Khá khó hiểu đúng không, này có dịp mình sẽ giải thích kĩ hơn nguyên lý này. Còn tại sao nguyên lý này lại nổi tiếng và có sức mạnh như vậy thì mình sẽ minh hoạ qua một ví dụ đơn giản như sau:

Dưới đây là ví dụ về cách sử dụng raw pointer để quản lý bộ nhớ:

```cpp
#include <iostream>
using namespace std;

void rawPointerExample() {
    int* rawPtr = new int(10);  // Cấp phát bộ nhớ động
    cout << "Raw Pointer Value: " << *rawPtr << endl;
    delete rawPtr;             // Phải giải phóng thủ công
}

```

Còn đây là sức mạnh khi sử dụng smart pointer:

```cpp
#include <iostream>
#include <memory> // Thư viện cho smart pointers
using namespace std;

void smartPointerExample() {
    unique_ptr<int> smartPtr = make_unique<int>(10);  // Tạo smart pointer
    cout << "Smart Pointer Value: " << *smartPtr << endl;
    // Không cần gọi delete, bộ nhớ sẽ được giải phóng tự động.
}
```

Bạn thấy sao khi mình không cần phải quản lý thủ công về memory trên bộ nhớ Heap nữa. Sức mạnh của RAII ở đây chính là việc mình gắn liền vòng đời bộ nhớ được tạo ở Heap thông qua vòng đời của smartPtr chứ không còn là thủ công delete như trước nữa.

### Các loại Smart pointer

Trong C++ STL, có ba loại smart pointers chính là `std::unique_ptr`, `std::shared_ptr`, và `std::weak_ptr`. 

1. `std::unique_ptr`
    - `std::unique_ptr` đại diện cho **quyền sở hữu duy nhất** (*unique ownership*) của một tài nguyên.
    - Không cho phép chia sẻ quyền sở hữu.
    - Khi một `std::unique_ptr` bị hủy hoặc gán cho con trỏ khác, tài nguyên của nó sẽ được giải phóng tự động.
    
    **Khi nào nên sử dụng?**
    
    - Khi bạn cần quản lý một đối tượng mà **chỉ một thành phần duy nhất** sở hữu và quản lý nó.
    - Đối tượng này không được chia sẻ giữa nhiều thành phần trong chương trình.
    
    ```cpp
    void uniquePtrExample() {
        auto uniquePtr = make_unique<int>(42); // Tạo unique_ptr
        cout << "Value managed by unique_ptr: " << *uniquePtr << endl;
    
        // uniquePtr không thể sao chép (copy) hoặc chia sẻ
        // auto anotherPtr = uniquePtr; // Lỗi: Không cho phép copy
    }
    ```
    
2. `std::shared_ptr`
    - `std::shared_ptr` đại diện cho **quyền sở hữu được chia sẻ** (*shared ownership*).
    - Nhiều `std::shared_ptr` có thể trỏ đến cùng một tài nguyên, và tài nguyên đó chỉ được giải phóng khi không còn `std::shared_ptr` nào trỏ đến nó.
    - Sử dụng cơ chế **reference counting** (đếm tham chiếu) để theo dõi số lượng con trỏ tham chiếu đến tài nguyên.
    
    ![How shared_ptr work [3]](./img/shared_ptr.webp "How shared_ptr work [3]")

    **Khi nào nên sử dụng?**
    
    - Khi tài nguyên cần được chia sẻ giữa nhiều thành phần hoặc luồng trong chương trình (do biến count trong `shared_pointer` là atomic).
    - Phù hợp với các đối tượng có vòng đời phức tạp mà nhiều thành phần đều cần truy cập.
    
    ```cpp
    void sharedPtrExample() {
        auto shared1 = make_shared<int>(42); // Tạo shared_ptr đầu tiên
        {
            auto shared2 = shared1; // Chia sẻ quyền sở hữu
            cout << "Value managed by shared_ptr: " << *shared2 << endl;
            cout << "Reference count: " << shared1.use_count() << endl; // 2 references
        }
        // Khi shared2 ra khỏi phạm vi, reference count giảm xuống 1
        cout << "Reference count after scope: " << shared1.use_count() << endl; // 1 reference
    }
    ```
    
    **Lưu ý khi sử dụng:**
    
    Có thể xảy ra tình trạng vòng lặp tham chiếu như sau:
    
    ```cpp
    class Child; // Khai báo trước
    
    class Parent {
    public:
        shared_ptr<Child> child; // Parent trỏ đến Child
        ~Parent() {
            cout << "Parent destroyed" << endl;
        }
    };
    
    class Child {
    public:
        weak_ptr<Parent> parent; // Child trỏ ngược đến Parent bằng weak_ptr
        ~Child() {
            cout << "Child destroyed" << endl;
        }
    };
    
    void fixedCircularReference() {
        auto parent = make_shared<Parent>();
        auto child = make_shared<Child>();
    
        parent->child = child;  // Parent trỏ đến Child
        child->parent = parent; // Child trỏ ngược lại Parent bằng weak_ptr
    
        // Khi hàm kết thúc, cả parent và child sẽ được giải phóng.
    }
    ```
    
    Để giải quyết tình trạng trên, C++ giới thiệu loại con trỏ thứ 3 là weak_pointer
    
3. `std::weak_ptr`
    - `std::weak_ptr` không sở hữu tài nguyên mà chỉ giữ một **tham chiếu yếu** đến tài nguyên được quản lý bởi `std::shared_ptr`.
    - Được thiết kế để tránh vòng tham chiếu khi hai hoặc nhiều `std::shared_ptr` trỏ lẫn nhau.
    - Không tăng reference count của tài nguyên.
    
    **Khi nào nên sử dụng?**
    
    - Khi cần trỏ đến một tài nguyên được quản lý bởi `std::shared_ptr` nhưng không muốn ảnh hưởng đến vòng đời của nó.
    - Thường dùng trong các cấu trúc như cây cha-con hoặc đồ thị, nơi các tham chiếu lẫn nhau có thể xảy ra.
    
    Ví dụ về cách fix bug circular references mà mình giới thiệu ở trên
    
    ```cpp
    class Child; // Khai báo trước
    
    class Parent {
    public:
        shared_ptr<Child> child; // Parent trỏ đến Child
        ~Parent() {
            cout << "Parent destroyed" << endl;
        }
    };
    
    class Child {
    public:
        weak_ptr<Parent> parent; // Child trỏ ngược đến Parent bằng weak_ptr
        ~Child() {
            cout << "Child destroyed" << endl;
        }
    };
    
    void fixedCircularReference() {
        auto parent = make_shared<Parent>();
        auto child = make_shared<Child>();
    
        parent->child = child;  // Parent trỏ đến Child
        child->parent = parent; // Child trỏ ngược lại Parent bằng weak_ptr
    
        // Khi hàm kết thúc, cả parent và child sẽ được giải phóng.
    }
    
    ```
    

### Bảng so sánh 3 loại smart pointer

| **Thuộc tính** | **std::unique_ptr** | **std::shared_ptr** | **std::weak_ptr** |
| --- | --- | --- | --- |
| **Quyền sở hữu** | Duy nhất (*unique*) | Chia sẻ (*shared*) | Không sở hữu (*non-owning*) |
| **Cơ chế quản lý** | Giải phóng ngay khi hết phạm vi. | Đếm tham chiếu (*reference counting*). | Dựa vào tài nguyên `std::shared_ptr`. |
| **Chi phí quản lý** | Thấp | Cao hơn do reference counting. | Thấp (không tăng reference count). |
| **Sử dụng khi nào?** | Khi tài nguyên chỉ được một đối tượng quản lý. | Khi cần chia sẻ tài nguyên giữa nhiều thành phần. | Khi cần tham chiếu yếu để tránh vòng lặp. |

### Sử dụng smart pointer sao cho đúng cách?

Smart pointers là một công cụ mạnh mẽ trong C++, nhưng để sử dụng hiệu quả và tránh những vấn đề không mong muốn, bạn cần nắm rõ một số lưu ý sau:

1. **Sử dụng `std::make_shared` thay vì khởi tạo trực tiếp**

**`std::make_shared`** là cách khởi tạo `std::shared_ptr` nên sử dụng vì nó mang lại hiệu suất cao và an toàn hơn so với việc sử dụng `new` trực tiếp.

**Lợi ích của `std::make_shared`**

- **Hiệu suất tốt hơn**:`std::make_shared` thực hiện một lần cấp phát bộ nhớ cho cả đối tượng và bộ đếm tham chiếu, trong khi khởi tạo trực tiếp với `new` cần hai lần cấp phát.
- **An toàn hơn**:Tránh các lỗi khi ngoại lệ xảy ra giữa quá trình cấp phát tài nguyên và khởi tạo `shared_ptr`.

```cpp
// Khởi tạo trực tiếp với new 
void directInitialization() {
    shared_ptr<int> sptr(new int(42)); // Tốn thêm chi phí cấp phát cho count reference
    cout << "Value: " << *sptr << endl;
}

// Khởi tạo với make_shared (khuyến nghị)
void makeSharedExample() {
    auto sptr = make_shared<int>(42); // Tối ưu hóa hiệu suất
    cout << "Value: " << *sptr << endl;
}
```

**2. Tránh vòng tham chiếu với `std::weak_ptr`**

**Vòng tham chiếu** thường xảy ra khi hai `std::shared_ptr` tham chiếu lẫn nhau, làm tăng reference count và ngăn tài nguyên được giải phóng. Để tránh vấn đề này, hãy sử dụng `std::weak_ptr` cho một trong các tham chiếu.

**Mẹo sử dụng `std::weak_ptr`:**

- Dùng `std::weak_ptr` để trỏ đến tài nguyên mà bạn **không sở hữu**.
- Kiểm tra tài nguyên còn tồn tại trước khi sử dụng bằng cách gọi `expired()` hoặc chuyển thành `shared_ptr` bằng `lock()`.

**3. Cân nhắc giữa việc sử dụng các loại smart pointer**

Trong các ứng dụng phức tạp hoặc có yêu cầu hiệu suất cao, việc sử dụng smart pointers cần được cân nhắc kỹ lưỡng để tránh các chi phí không cần thiết.

**Ví dụ cụ thể:**

- **Ưu tiên sử dụng `std::unique_ptr` khi có thể**:
    - `std::unique_ptr` không có chi phí quản lý reference count như `std::shared_ptr`.
    - Dùng cho các đối tượng chỉ có **một quyền sở hữu duy nhất**.
- **Hạn chế sử dụng `std::shared_ptr` trong các vòng lặp lớn**:
    - Việc tăng và giảm reference count liên tục trong các vòng lặp tính toán có thể gây tốn hiệu năng.

```cpp
void efficientUniquePtrUsage() {
    vector<unique_ptr<int>> vec;
    for (int i = 0; i < 100000; ++i) {
        vec.push_back(make_unique<int>(i)); // Không có chi phí reference count
    }
}
```

### Kết luận

Smart pointers là một trong những cải tiến mạnh mẽ nhất trong `C++11` để quản lý tài nguyên một cách hiệu quả và an toàn. Chúng không chỉ giúp lập trình viên tránh được những lỗi phổ biến như rò rỉ bộ nhớ hay con trỏ treo, mà còn làm cho mã nguồn dễ bảo trì hơn và tương thích tốt với các ứng dụng hiện đại.

Hiện tại khi mình lập trình với những project mới thì 100% các project mới đều sử dụng smart pointer thay vì raw pointer thông thường. Đối với bản thân mình smart pointer là một phần không thể thiếu trong các dự án C++ hiện đại. Nhưng việc sử dụng chúng đúng cách đòi hỏi sự hiểu biết sâu sắc và kỹ năng thực hành tốt.

### **Tài liệu tham khảo**

1. [**CppReference**:](https://en.cppreference.com/book/intro/smart_pointers)
2. [Smart Pointers in C++ - GeeksforGeeks](https://www.geeksforgeeks.org/smart-pointers-cpp/)
3. [C++ Smart Pointer Explained Through Intuitive Visuals | by Joseph Robinson, Ph.D. | Better Programming](https://betterprogramming.pub/understanding-smart-pointer-iii-909512a5eb05)