# Visualization of Rust Lifetime Parameter

## Lifetime Parameter - An Overview

Lifetime parameters are a specialty of the Rust language. They're mainly used by the borrow checker to make sure usage of references is safe, meaning they simply follow one rule: a reference cannot outlive the original variable it's pointing to. Lifetime parameters typically annotate the lifetime relation between:

+ References passed to a function call, and possibly the return value as well.

+ References inside a struct and the struct variable upon creation.

The borrow checker will keep a table of lifetimes for all variables, either by looking at the scope (line number) or via the aid of lifetime parameters. If a reference is accessed outside its scope (the lifetime looked up by the borrow checker), then an error will be issued ([see how the borrow checker reports reference errors](https://alaric617r.github.io/Rust-Blog/Intro%20to%20Polonius.html)).

## Lifetime Parameter in Normal Functions

Let's look a simple function that takes two references to `i32` and returns the reference to the bigger one:

```rust
fn max<'a>(x: &'a i32, y: &'a i32) -> &'a i32{
    if x >= y{
        x
    }
    else{
        y
    }
}
```

And a caller for `max`:

```rust
fn main(){
#1    let a = 10;
#2    let b = 6;
#3    let r: &i32;
#4    {
#5        let x: &i32 = &a;
#6        let y: &i32 = &b;
#7        r = max(x,y);
#8    }
#9    println!("r is {}",r);
#10 }
```

To reason how borrow checker calculates `'a` of `max`, let's first draw out the lifetime of each variable in the caller's point of view: 

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/max_var_lifetime.jpeg)

Then, the lifetime parameter can be easily computed just by looking at the function signature of `max`:

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/max_lp.jpeg)

`'a` should be able to encompass the lifetime of all three references and be as small as possible. In this case, `'a = [#3, #9]`, the same as the lifetime of `r`. Otherwise, the borrow checker will turn down even the correct usage of some references, since the lifetime assigned to that reference ends earlier than it should. Say the borrow checker assigns `'a = [#3, #8]`, then it will turn down `println!("r is {}",r);` on line 9, since `r` should be dead on line 8. But it's obvious that the borrow checker is being over-protective, since accessing the path of `r` is totally safe, as `r` is pointing to one of `a` and `b` and both of these variables still have life on line 9.

## Lifetime Parameter in Structs

A struct must have explicit lifetime parameter annotations if it contains a reference. The reason is simple: since the reference can't outlive the lender variable, the struct also cannot. Lifetime parameters are the way to correlate the lifetime of the reference and the struct.

Let's see how the lifetime parameter is computed when used to annotate a struct:

```rust
struct Book<'a>{
    name: &'a String,
    serial_num: i32
}

impl<'a> Book<'a>{
    fn new(_name: &'a String, _serial_num: i32) -> Book<'a>{
        Book { name: _name, serial_num: _serial_num }
    }
}
```

Here we define a `Book` struct that contains an immutable reference to a `String`. `'a` means that if the lifetime of `Book` is `'a`, then the lifetime of `name : &'a String` will be at least `'a`.

The `impl` block contains one function that create `Book` type in a factory mode. Let's look at an example of calling this function:

```rust
fn main(){
#1    let mut name = String::from("The Rust Book");
#2    let serial_num = 1140987;
#3    {
#4        let rust_book = Book::new(&name, serial_num);
#5        println!("The name of the book is {}",rust_book.name);
#6    }
#7    name = String::from("Behind Borrow Checker");
#8    println!("New name: {}",name);
#9 }
```

First and foremost, let's calculate the lifetime of each variable (this will always be the first step, so bear in mind). It suffices to just look at line numbers and scoping curly brackets:

| Variable     | Lifetime |
|:------------:|:--------:|
| `name`       | [#1, #9] |
| `serial_num` | [#2, #9] |
| `rust_book`  | [#4, #6] |

Note that since `rust_book` is in an inner scope, it will be destructed when the scope ends. To calculate lifetime parameter on `Book::new()` invoked on line 4, we list the lifetime of all variables engaged:

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/book_life_viz.png)

Since `&name` is a temporary variable created just to be passed on line 4, its lifetime will be limited only to line 4. As `'a` should encompass the lifetime of references both in `rust_book.name`, as well as `&name`, `'a` = [#4, #6]. This calculation is passed to `struct Book<'a>`, so the lifetime parameter of `rust_book.name` will be [#4,#6].

Note that this program runs well because every usage of a reference is within its legal lifetime scope. For example, changing `name` on line 7 won't cause a `value still borrowed` error since `rust_book.name`, which is a reference to `name`, has been out of scope since line 6 based on the lifetime we calculated.

##### About reference passed on the fly

You may question why `&name` is only live on line 4. We could imagine  the borrow checker will create a reference to `name` and pass that reference to the function, and everything is happening on a single line:

```rust
#4 let tmp = &name;
#4 let rust_book = Book::new(tmp, serial_num);
#4 // tmp's lifetime is end here
```

As [NLL lifetime](https://stackoverflow.com/questions/50251487/what-are-non-lexical-lifetimes) defines a reference's lifetime ends after its last usage, `tmp` is created on line 4 and also ended on the same line. We will treat lifetime of references created on the fly in this way in the following tutorial.

## Lifetime Parameter - A More Complex Example

Previous examples have equipped you with the basic methodology for reasoning out generic lifetime parameters, just as the borrow check does. In this section, we will use lifetime parameters to construct a real-life problem-solver: a scheduler program that processes requests in a database. Let's dive in.

### Modeling Requests Using Structs

First, we need to prototype incoming requests for our database. For simplicity, the DB system only supports four kinds of operations: `Create`, `Read`, `Update`, and `Delete`, or [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete). Each kind of request will be issued in batches by different users. To model that, we can define an `enum` of all CRUD request types:

```rust
enum RequestType{
    CREATE,
    READ,
    UPDATE,
    DELETE,
}

impl RequestType {
    fn to_string(&self) -> String{
        match self {
            RequestType::CREATE => String::from("create"),
            RequestType::READ => String::from("read"),
            RequestType::UPDATE => String::from("update"),
            RequestType::DELETE => String::from("delete"),
        }
    }
}
```

We can also add a simple `impl` function `to_string()` for easy logging. Then, our `Request` struct should contain:

+ `request_type: RequestType`, which is the type of CRUD request

+ `num_request_left: &mut u32`, the number of CRUD requests in a batch. A mutable reference to `u32` means if our database can't handle a full batch of request, we will process as many as we can and the mutable reference records how many requests are served.

```rust
struct Request<'a>{
    num_request_left: &'a mut u32,
    request_type: RequestType
}

impl<'a> Request<'a>{
    fn new(_num_request: &'a mut u32, _request_type: RequestType) -> Request<'a>{
        Request { num_request_left: _num_request, request_type: _request_type }
    }
}
```

Note that since `struct Request` contains a reference, we must make sure `Request` instance doesn't outlive the lender of `num_request_left`, meaning an explicit lifetime parameter is needed. For convenience, we also define a `Request::new()` function to create `Request` in factory mode. The rational behind the lifetime annotation has already been explained in the previous section.

#### Data Structure for Our Request Queue: `VecDeque`

`std::collections::VecDeque` is just like `std::deque<T>` in C++, which can either be FIFO or FILO order. We will choose FIFO to process the incoming requests. Let's first get familiar with this data structure with two examples:

+ Correct usage:

```rust
use std::collections::VecDeque;

#1 fn main(){
#2    let mut queue:VecDeque<&i32> = VecDeque::new();
#3    let a = 10;
#4    let b = 9;
#5    queue.push_back(&a);
#6    queue.push_back(&b);
#7    loop {
#8        if let Some(num) = queue.pop_front(){
#9            println!("element: {}",num);
#10        }
#11        else {
#12           break;
#13        }
#14    }
#15 }
```

Here, we create a `VecDeque` to store `&i32`, because later we're going to store a reference to `Request`, so it's beneficial to find some insights on using references as stored data beforehand. On lines 5-6, we push references of `a` and `b` into the queue. Then, we loop through the queue in FIFO order by calling `pop_front()`. Note that unlike C++, the return type of `pop_front()` is not `&i32` but `Option<&i32>`:

> `alloc::collections::vec_deque::VecDeque`
> 
> `pub fn pop_front(&mut self) -> Option<T>`
> 
> ---
> 
> Removes the first element and returns it, or `None` if the deque is empty.

This provides a safety net if the user calls `pop_front()` when `VecDeque` is empty. To inspect whether `queue` is empty or not, we use `if let` syntax to extract the value stored in the front of queue,  and break if we see a `Option::None`.

+ Erroneous Usage:

```rust
#1 fn main(){
#2    let mut queue:VecDeque<&i32> = VecDeque::new();
#3    {
#4        let a = 10;
#5        let b = 9;
#6        queue.push_back(&a);
#7        queue.push_back(&b);
#8    }
#9    let c = 6;
#10   queue.push_back(&c);
    // error: `a` does not live long enough
    // label: borrow later used here
    // error: `b` does not live long enough
    // label: borrow later used here
#11 }
```

The borrow checker will complain at line 10 that `a` and `b` don't live long enough. This is obvious, since `a` and `b` are in the inner scope which ends at line 8. However, references to `a` and `b` are still inside `queue` till line 10. This will cause an invalid read to deallocated stack space, which the borrow check will prevent during compile time.

Therefore, an important insight into a `Vecdeque` holding references is that *the lifetime of `VecDeque` is bounded by the shortest lifetime among all references it stores*. This can be explained by the  [lifetime elision rule](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-elision) on method definition.

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-12-46-13-image.png)

Since `a` and `b` only live from lines 4/5 to line 8, the lifetime of `queue` lasts at most until line 8. So if we remove the stuff after line 9, the borrow check will let us pass.

### Process DB Requests

We're so close to our goal! Let's design how our database handles user requests. It's typical that there will be multiple threads inside a database that serve requests concurrently, and each thread grabs some resources from a pool. We simplify each thread into a function that takes a queue of `Request`, and the number of resources it possesses, `max_process_unit`, meaning the maximum number of CRUD operations it can perform:

```rust
/// process requests passed from a mutable deque, return the front request if it hasn't been fully served
///
/// # Arguments
///
/// * 'queue' - A mutable reference to deque which stores requests in FIFO order, requests are mutable reference to Request struct type
///
/// * 'max_process_unit' - A mutable reference to maximal number of operation (READ, UPDATE, DELETE) it can serve
///
fn process_requests<'i>(queue: &'i mut VecDeque<&'i mut Request<'i>>, max_process_unit: &'i mut u32) -> Option<&'i mut Request<'i>>{
    loop {
        let front_request: Option<&mut Request> = queue.pop_front();
        if let Some(request) = front_request{
            // if current max_process_unit is greater than current requests
            if request.num_request_left <= max_process_unit{
                println!("Served #{} of {} requests.", request.num_request_left, request.request_type.to_string());
                // decrement the amount of resource spent on this request
                *max_process_unit = *max_process_unit - *request.num_request_left;
                // signify this request has been processed
                *request.num_request_left = 0;
            }
            // not enough
            else{
                // process as much as we can
                *request.num_request_left = *request.num_request_left - *max_process_unit;
                // sad, no free resource anymore
                *max_process_unit = 0;
                // enqueue the front request back to queue, hoping someone will handle it...
                // queue.push_front(request);
                return Option::Some(request);
            }
            //
        }
        else {
            // no available request to process, ooh-yeah!
            return Option::None;
        }
    }
}
```

We won't hasten to explain how lifetime parameter works here; instead, we will embed it in some caller. For now, just understand what it wants to accomplish:

1. First, loop through a batch of requests in FIFO order;

2. When `queue` is not empty, serve the front request as much as we can. 
   
   + If `max_process_unit` is enough to cover this request, just decrements `max_process_unit` by `request.num_request_left` and keeps processing;
   
   + If `max_process_unit` is not enough to complete the whole request, we will return the request reference and hope some other handler will process it, or just put it back to the queue, which is out of our concern.

3. If `queue` is empty, it means all requests have been served and we return blissfully. 

### Finally Inside the Database

Eventually, we're able to put together all the pieces and design our DB. For simplicity, we will model this as single-threaded:

```rust
fn main() {
#1     let mut request_queue: VecDeque<&mut Request> = VecDeque::new();
#2     // generating some requests
#3     let mut reads_cnt: u32 = 20;
#4     let mut read_request: Request = Request::new(&mut reads_cnt, RequestType::READ);
#5     // enqueue read requests
#6     request_queue.push_back(&mut read_request);
#7     let mut updates_cnt: u32 = 30;
#8     let mut update_request: Request = Request::new(&mut updates_cnt, RequestType::UPDATE);
#9     request_queue.push_back(&mut update_request);
#10    // enqueue update requests
#11    let mut deletes_cnt: u32 =50;
#12    let mut delete_request: Request = Request::new(&mut deletes_cnt, RequestType::DELETE);
#13    request_queue.push_back(&mut delete_request);
#14    // enqueue delete requests
#15    // something else happening in DB...
#16   // ..., process requests
#17    let mut available_resource: u32 = 10;
#18    let request_halfway : Option<mut Request> = process_requests(&mut request_queue, &mut available_resource);
#19    if let Some(req) = request_halfway {
#20        println!("#{} of {} requests are left unprocessed!", req.num_request_left, req.request_type.to_string());
#21    }
#22    println!("there are #{} free resource left.", available_resource);
#23 }
```

Let's first calculate the lifetime of each *variable*, since `Request` contains lifetime parameter itself.

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-15-26-03-image.png)

By previous section, it's easy to verify lifetime via function signature of `Request::new()` function. Therefore, we obtain the following table of variable lifetime:

*lifetime table, can integrate into visualization, now just for reasoning*

| variable                                    | lifetime (scope) |
|:-------------------------------------------:|:----------------:|
| `mut request_queue: VecDeque<&mut Request>` | [#1, #23]        |
| `mut reads_cnt : u32`                       | [#3, #23]        |
| `mut read_request: Request`                 | [#4, #23]        |
| `mut updates_cnt : u32`                     | [#7, #23]        |
| `mut update_request: Request`               | [#8, #23]        |
| `mut deletes_cnt : u32`                     | [#11, #23]       |
| `mut delete_request: Request`               | [#12, #23]       |
| `mut available_resource: u32`               | [#17, #23]       |
| `request_halfway : Option<mut Request>`     | [#18, #23]       |

We called `process_requests()` on line 18 --- let's have a closer look how the lifetime parameter is calculated in this case by inspecting the function signature:

```rust
fn process_requests<'i>(queue: &'i mut VecDeque<&'i mut Request<'i>>, max_process_unit: &'i mut u32) -> Option<&'i mut Request<'i>>{
```

This function signature is a bit terrifying, with references and `'i`s all over the place. To make it a bit easier, we'll break it down into three distinct parts, which we'll call `'i1`, `'i2`, and `'i3`. Each is a different reference within the total lifetime that makes up `'i`. 

+ `'i1`: `queue: &'i1 mut VecDeque<&'i1 mut Request<'i1>>`

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-17-01-18-image.png)

From the graph, we see the references `request_queue` stores are still alive till line 20. That's because the [NLL (Non-lexical lifetime)](https://stackoverflow.com/questions/50251487/what-are-non-lexical-lifetimes) defines a reference as alive until it will not be used anymore. In this case, `req` (Note that `req` is of type `&mut Request`), which is defined by `if let Some(req) = request_halfway` can be any one of these references (i.e, `&read_request`, `&update_request`, `&delete_request`) if the `if let` statement is true. The borrow checker will be a protective to all control flow possibilities, so the last use of any reference to R/U/D will be line 20. 

Therefore, `'i1 = [#18, #20]`.

+ `'i2`: `max_process_unit: &'i2 mut u32` 

This is easy, since `&available_resource` is passed at line 18, and since it's a temporary reference, it ends on line 18 as well. So `'i2 = [#18, #18]`.

+ `'i3`: `-> Option<&'i mut Request<'i3>>`

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-16-49-03-image.png)

This case is already contained within `'i1`. If the `if let` statement is true, data stored inside `request_halfway` will be moved into `req`, which results in the end of lifetime of `request_halfway`. So `'i3 = [#18, #19]`

Combining `'i1`, `'i2` and `'i3`, we have finally calculated `'i`, which encompasses all three sub-lifetimes to be as small as possible. Think of it this way: the compiler wants to know, of the references that are passed in and the reference that is returned, what is the lifetime it needs to make sure all are valid when it returns? By marking them all with `'i`, we are telling the compiler that the return reference will be valid as long as any reference we give it is valid. Hence, `'i = [#18, #20]`. At last, we've made sure that all references' liftetimes pass the borrow checker. Hooray!

### 
