# Visualization of Rust Lifetime

## Lifetime - An Overview

It's Rust borrow checker's good intention to eradicate undefined and unsafe behavior at compile time. One good object to look at is the usage of references, since unsafe access to memory will occur if a reference lives longer than the original lender of the resource. However, borrow checker has loosen the constraint on programmers, with the introduction of NLL lifetime (check this [RFC](https://rust-lang.github.io/rfcs/2094-nll.html)). However, we will not focus on NLL lifetime, but how to visualize lifetime and lifetime parameter, with the latter tends to be a common headache for most rust rookies。

## Lifetime Parameter in Normal Functions

Let's look a simple function that takes two reference to `i32` and return the reference to the bigger one:

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

To reason how borrow checker calculates `'a` of `max`, let's first draw out the lifetime of each variable in caller's point of view: 

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/max_var_lifetime.jpeg)

Then, lifetime parameter could be easily computed just looking at the function signature of `max`:

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/max_lp.jpeg)

`'a` should be able to encompass the lifetime of all three reference and be as small as possible, which in this case, `'a = [#3, #9]`, the same as lifetime of `r`. Otherwise, borrow checker will turn down even correct usage of some reference, since the lifetime assigned to that reference ends earlier than it should be. Say the borrow checker assign `'a = [#3, #8]`, then it will turn down `println!("r is {}",r);` on line 9, since `r` should be dead on line 8. But it's obvious that the borrow checker is being over-protective, since accessing the path of `r` is total safe, since `r` is pointing to one of `a` and `b` and both of these variables are life on line 9.

## Lifetime Parameter in Structs

A struct must have explicit lifetime parameter as annotation if it contains a reference. The reason is simple, since the reference can't outlive the lender variable, the struct also cannot. Lifetime parameter is the way to correlate the lifetime of the reference and the struct.

Let's see how lifetime parameter is computed when used to annotate a struct:

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

The `impl` block contains one function that create `Book` type in a factory mode. Let's see an example of calling this function:

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

First and foremost, let's calculate the lifetime of each variable (this will always be the first step, so bear in mind). It suffices just look at line numbers and scoping curly brackets:

| Variable     | Lifetime |
|:------------:|:--------:|
| `name`       | [#1, #9] |
| `serial_num` | [#2, #9] |
| `rust_book`  | [#4, #6] |

Note that since `rust_book` is in an inner scope, it will be destructed when the scope ends. To calculate lifetime parameter on `Book::new()` invoked on line 4, we list the lifetime of all variables engaged:

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/book_life_viz.png)

Since `&name` is a temporary variable just passed on line 4, its lifetime will just be limited in line 4. As `'a` should encompass reference lifetime both in `rust_book.name` as well as `&name`,`'a` = [#4, #6]. This calculation also get passed to `struct Book<'a>`, so the lifetime parameter of `rust_book.name` will be [#4,#6].

Note that this program runs well because every usage of reference is in its legal lifetime scope. For example, changing `name` on line 7 won't cause a `value still borrowed` error since `rust_book.name`, which is a reference to `name`, has been out of scope on line 6 based on the lifetime we calculated.

## Lifetime Parameter - A More Complex Example

Previous examples have equipped you with the basic methodology into reasoning generic lifetime parameter, just as the borrow check does. In this section, we will use lifetime parameter to construct a real-life problem-solver: a scheduler program that process requests in a database. Let's dive in.

### Modeling Requests Using Structs

First, we need to prototype incoming request for our database. For simplicity, the DB system only supports four kinds of operation:

`Create`, `Read`,`Update`,`Delete`, namely [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete). Each kind of request will be issued in batches by different users. To model that, we can do it via defining an `enum` of all CRUD request:

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

Note that since `struct Request` contains a reference, we must make sure `Request` instance doesn't outlive the lender of `num_request_left`, meaning an explicit lifetime parameter is needed. For convenience, we also define a `Request::new()` function to create `Request` in factory mode. The rational behind the lifetime annotation has been already explained in previous section.

#### Data Structure for Our Request Queue: `VecDeque`

`std::collections::VecDeque` is just like `std::dequeu<T>` in C++, which can either be FIFO or FILO order. We will choose FIFO to process the incoming request. Let's first get familiar with this data structure by two examples:

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

Here, we create a `VecDeque` to store `&i32`, because later we're going to store reference to `Request` so it's beneficial to find some insights of using reference as data beforehand. On line 5-6, we push reference of `a` and `b` to the queue. Then, we loop through the queue in FIFO order by calling `pop_front()`. Note that difference from C++, the return type of `pop_front()` if not `&i32` but `Option<&i32>`:

> `alloc::collections::vec_deque::VecDeque`
> 
> `pub fn pop_front(&mut self) -> Option<T>`
> 
> ---
> 
> Removes the first element and returns it, or `None` if the deque is empty.

This provides safety each if the user calls `pop_front()` when `VecDeque` is empty. To inspect whether `queue` is empty or not, we use `if let` syntax to extract the value stored in the front of queue  and break if we see a `Option::None`.

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

The borrow checker will complain at line 10 that `a`, `b` don't live long enough. This is obvious since `a`,`b` are in a inner scope, which ends at line 8. However, references to `a` and `b` are still inside `queue` till line 10. This will cause invalid read to deallocated stack space, which the borrow check will prevent during compile time.

Therefore, an important insight into `Vecdeque` holding references is that *the lifetime of `VecDeque` is bounded by the shortest lifetime among all references it stores*. This can be explained by lifetime [elision rule](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-elision) on method definition.

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-12-46-13-image.png)

Since `a` and `b` only live from ln4/ln5 to ln8, lifetime of `queue` is at most till ln8. So if we remove the stuff after ln9, the borrow check will let us pass.

### Process DB Requests

We're so close to our goal! Let's design how our database handles user request. It's usual that there will be multiple threads inside a DB that serve requests concurrently and each thread grab some resources from a pool. We simplify each thread into a function that takes a queue of `Request` and the number of resources it possess `max_process_unit`, meaning the maximum number of CRUD operations it can perform:

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

We won't haste to explain how lifetime parameter works here, instead we will embed it with some caller. For now, just understand what it wants to accomplish:

1. First, loop through batched requests in FIFO order;

2. When queue not empty, serve the front request as much as we can. 
   
   + If `max_process_unit` is enough to cover this request, just decrements `max_process_unit` by `request.num_request_left` and keeps processing;
   
   + If `max_process_unit` is not enough to complete the whole request, we will return the request reference and hopes some handler will process it, or just put it back to the queue, which is out of our concern.

3. If queue is empty, it means all requests have been served and we return blissfully. 

### Inside the DB Finally

Eventually, we're able to put together all the pieces and design our DB. For simplicity, we model this as single-threaded:

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

We called `process_requests()` on ln18, lets have closer look how lifetime parameter is calculated in this case by inspecting the function signature:

```rust
fn process_requests<'i>(queue: &'i mut VecDeque<&'i mut Request<'i>>, max_process_unit: &'i mut u32) -> Option<&'i mut Request<'i>>{
```

It's a bit terrifying as there are a lot of type specifying and lifetime annotations, so we'll break down by parts:

+ `queue: &'i1 mut VecDeque<&'i1 mut Request<'i1>>`

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-17-01-18-image.png)

From the graph, we see the reference `request_queue` stores are still alive till #20. That's because the [NLL lifetime](https://stackoverflow.com/questions/50251487/what-are-non-lexical-lifetimes) defines a reference is alive until it will not be used anymore. In this case, `req` (Note that `req` is of type `&mut Request`), which is defined by `if let Some(req) = request_halfway` can be any one of these references (i.e, `&read_request`, `&update_request`, `&delete_request`) if the `if let` statement is true. The borrow checker will be a protective to all control flow possibilities, so that the last use of any reference to R/U/D will be line 20. 

Therefore,`'i1 = [#18, #20]`.

+ `max_process_unit: &'i2 mut u32` 

This is easy, since `&available_resource` is passed at line 18 and since it's a temporary reference, it ends on line 18 as well. So `'i2 = [#18, #18]`.

+ `-> Option<&'i mut Request<'i3>>`

![](/Users/alaric66/Desktop/research/RustViz/Rust-blogs/2023-06-04-16-49-03-image.png)

This case is already contained in `'i1`. If the `if let` statement is true, data stored inside `request_halfway` will be moved into `req`, which results in the end of lifetime of `request_halfway`.So `'i3 = [#18, #19]`

Combining `'i1`,`'i2` and `'i3`, we finally calculated `'i`, which shall encompass all three sub-lifetimes and be as small as possible. Hence, `'i1` = [#18, #20]. Eventually, all references' lifetimes are made sure to pass the borrow checker, Hooray!

### 
