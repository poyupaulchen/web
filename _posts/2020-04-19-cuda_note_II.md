---
layout: post
title: Notes on CUDA programming II
subtitle: Matrix multiplication
tags: [programming, GPU]
author: Yuanpeng Zhang
comments: true
use_math: true
---

<p style='text-align: justify'>
First of all, we may need a review for the very basic CUDA programming and GPU execution principle. Here, I want to use the diagram below to demonstrate the threadings on GPU with a very simple array addition operation example.
</p>

<p align='center'>
<img src="/assets/img/posts/GPU_Basic_Operation_Demo.png"
   style="border:none;"
   alt="gpu_demo_1"
   title="gpu_demo_1" />
<br />
</p>

<p style='text-align: justify'>
In this simple case, we only have a single block on GPU, which contains, say, 500 threads (the actual number of threads may be larger than 500 but in this example, we only need 500 of them). The part above the dashed line refers to the single block. The for loop in the middle of the bottom half is usually what we do for coding in the array addition task. Basically, we loop over all the 500 indexes to do the addition one-by-one in a serial manner. Using GPU, the working scheme is different. Instead of a single 'worker' executing our commands one after another, we now have a whole bunch of 'workers' ready at the same time, waiting for our commands so that they can do their own job, independently. Each 'worker' has his/her unique ID - the location within the grid-block-thread organization we talked about in previous note (<a target="_blank" href="https://draft.blogger.com/blog/post/edit/713170236114697752/7568363057236278083#">Click me</a> to transfer there). At the beginning of the workflow, we let each 'worker' shout out their unique ID ('threadIdx.x' in the diagram above). Then we decide which part of the job to assign to each specific 'worker' (the assignment 'i = threadIdx.x' in the diagram above). After that, each 'worker' knows exactly what he/she is responsible for and then goes directly to do job (e. g. 'c[35] = a[35] + b[35]' in the diagram above). Here, the diagram only shows the working mechanism on GPU and the data copying in between the host (CPU) and the device (GPU) is not shown. Basically, copying data (the array 'a' and 'b' here) from host to device is analogous to trucks delivering materials to the factory (the device, our GPU) and copying data (the resulted array 'c' here) from device back to host is analogous to factory delivering products to the boss (the host, CPU).

<br />

Since each 'worker' has his/her own unique ID with which we can assign a unique task for each of them. Therefore, the actual order of the job to be carried out does not really matter that much. That's why we see that in the diagram above, the #35 part of the job is shown above #10. This is just to demonstrate the idea that order here does not matter. In practice, they all happen in parallel and no one really knows or cares which one of them gets executed/finished first.

<br />

Based on the further discussion above, we now move on to a slightly complicated case, where we are going to do matrix multiplication with GPU. First, the basic principle of conducting matrix multiplication is shown in the following picture,
</p>

<p align='center'>
<img src="/assets/img/posts/GPU_Basic_Operation_Demo_2.png"
   style="border:none;"
   alt="gpu_demo_2"
   title="gpu_demo_2" />
<br />
</p>

<p style='text-align: justify'>
To conduct the matrix multiplication in a parallel way on GPU, the fundamental idea is to execute each of the block within dashed boxes, independently on individual thread on GPU. To realize that, the basic workflow is presented in the following picture,
</p>

<p align='center'>
<img src="/assets/img/posts/GPU_Basic_Operation_Demo_3.png"
   style="border:none;"
   alt="gpu_demo_3"
   title="gpu_demo_3" />
<br />
</p>

<p style='text-align: justify'>
Following the coding scheme as presented above, the fundamental difference as compared to usual serial coding here is that we represent the matrix in a one-dimensional manner, as indicated by the red arrows on the top of the figure above. The purpose of such a representation is for the ease of coding on GPU kernel, as we will see in the code snippet that will be presented later in current blog. Once we transform our matrix to its one dimensional form, the next step is again to let 'workers' shout out their unique ID, as we have discussed above. In this case, for demo purpose, assume we have a single block which contains \(3\times2\) thread structure, then in practice we have a one-to-one mapping between 'workers' and the entries of the final product matrix, i. e. each 'worker' is responsible for one entry in the final product matrix and none of them is wasted. The reason why we want to point out 'none is wasted' is that in practice, it often happens the overall GPU block unit is larger than what we really need. For example, in current example, if we claim a single GPU block with \(5\times4\) thread structure, we can still find right amount of 'workers' to do jobs for us, but in this case, for sure some of the 'workers' will not be assigned jobs, thus 'wasted'. Back to our workflow, once all 'workers' know what they are responsible for, they can go ahead to do their jobs, i. e. multiplying the right row and column of the input matrices and sum up the result to obtain the entry for the final product matrix, as indicated by the part of the diagram above containing a whole bunch of grouped (by 'worker') squares.

<br />

The explanation presented above about matrix multiplication with CUDA is based on the original blog post in Ref. [1]. Also, the code snippets there are reproduced here in current blog, with detailed comments for better understanding. A makefile, together with instructions about how-to, is also provided at the end of current blog, with which one can straightforwardly compile the code given below.
</p>

`main_mat_mul.cu`

<div align="right">
<button onclick="javascript:copytoclipboard('csp1')" style="border: none">Copy snippet to clipboard!</button>
</div>
<div style="background: rgb(255, 255, 255); border: solid gray; height: 1000px; overflow: auto; padding: 0.0em 0.0em; width: auto;"><table><tbody><tr><td><pre style="color: grey; line-height: 125%; margin-bottom: 0px; margin-right: 5px; margin-top: 0px;"> 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96</pre></td><td id="csp1"><pre style="line-height: 125%; margin: 0px;"><span style="color: #557799;">#include &lt;iostream&gt;</span>
<span style="color: #557799;">#include &lt;vector&gt;</span>
<span style="color: #557799;">#include &lt;stdlib.h&gt;</span>
<span style="color: #557799;">#include &lt;time.h&gt;</span>
<span style="color: #557799;">#include &lt;cuda_runtime.h&gt;</span>
<span style="color: #557799;">#include "kernel.h"</span>
<span style="color: #557799;">#include "kernel.cu"</span>
<span style="color: #557799;">#include "dev_array.h"</span>
<span style="color: #557799;">#include &lt;math.h&gt;</span>

using namespace std;

<span style="color: #333399; font-weight: bold;">int</span> <span style="color: #0066bb; font-weight: bold;">main</span>()
{
    <span style="color: #888888;">// Perform matrix multiplication C = A * B</span>
    <span style="color: #888888;">// where A, B and C are NxN matrices.</span>
    <span style="color: #333399; font-weight: bold;">int</span> N <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">16</span>;
    <span style="color: #333399; font-weight: bold;">int</span> SIZE <span style="color: #333333;">=</span> N<span style="color: #333333;">*</span>N;

    <span style="color: #888888;">// Allocate memory on the host.</span>
    vector<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> h_A(SIZE);
    vector<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> h_B(SIZE);
    vector<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> h_C(SIZE);

    <span style="color: #888888;">// Initialize matrices on the host.</span>
    <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> i<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; i<span style="color: #333333;">&lt;</span>N; i<span style="color: #333333;">++</span>){
        <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> j<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; j<span style="color: #333333;">&lt;</span>N; j<span style="color: #333333;">++</span>){
            h_A[i<span style="color: #333333;">*</span>N<span style="color: #333333;">+</span>j] <span style="color: #333333;">=</span> sin(i);
            h_B[i<span style="color: #333333;">*</span>N<span style="color: #333333;">+</span>j] <span style="color: #333333;">=</span> cos(j);
        }
    }

    <span style="color: #888888;">// Allocate memory on the device. The 'dev_array' refers to a class</span>
    <span style="color: #888888;">// defined separately in the 'dev_array.h' file - refer to the header</span>
    <span style="color: #888888;">// list on top of current file.</span>
    dev_array<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> d_A(SIZE);
    dev_array<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> d_B(SIZE);
    dev_array<span style="color: #333333;">&lt;</span><span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">&gt;</span> d_C(SIZE);

    <span style="color: #888888;">// The 'set' method defined in the 'dev_array.h' file is responsible for</span>
    <span style="color: #888888;">// copying data from host to device. Here, the first parameter '&amp;h_A[0]' or</span>
    <span style="color: #888888;">// '&amp;h_B[0]' is a pointer specifying the starting position in memory to</span>
    <span style="color: #888888;">// copy from. The second parameter 'SIZE' specifies the number of 'blocks'</span>
    <span style="color: #888888;">// we will copy. Detailed explanation about 'block' will be given in the </span>
    <span style="color: #888888;">// 'dev_array.h' file (refer to the definition of 'set' method there). </span>
    d_A.set(<span style="color: #333333;">&amp;</span>h_A[<span style="color: #0000dd; font-weight: bold;">0</span>], SIZE);
    d_B.set(<span style="color: #333333;">&amp;</span>h_B[<span style="color: #0000dd; font-weight: bold;">0</span>], SIZE);

    <span style="color: #888888;">// Now, we do the matrix multiplication on GPU, by calling the function</span>
    <span style="color: #888888;">// below, which is defined in another separate .cu file - 'kernel.cu' in</span>
    <span style="color: #888888;">// the header of current file. Within 'kernel.cu' file, the fundamental</span>
    <span style="color: #888888;">// kernel function will be defined and called, which is the real part that</span>
    <span style="color: #888888;">// is executing the matrix multiplication. Therefore, the calling scheme</span>
    <span style="color: #888888;">// can be roughly depicted as follows:</span>
    <span style="color: #888888;">//</span>
    <span style="color: #888888;">//                 call                        call</span>
    <span style="color: #888888;">// Current file   -----&gt; matrixMultiplication -----&gt; kernel</span>
    <span style="color: #888888;">// </span>
    matrixMultiplication(d_A.getData(), d_B.getData(), d_C.getData(), N);
    cudaDeviceSynchronize();

    <span style="color: #888888;">// The 'get' function is also defined in 'dev_array.h' file, as the 'set' </span>
    <span style="color: #888888;">// function above. It is responsible for copying data from device to host.</span>
    d_C.get(<span style="color: #333333;">&amp;</span>h_C[<span style="color: #0000dd; font-weight: bold;">0</span>], SIZE);
    cudaDeviceSynchronize();

    <span style="color: #888888;">// Next, we are going to do the same thing as above, i. e. matrix </span>
    <span style="color: #888888;">// multiplication. This time, we will be doing it on CPU only.</span>
    <span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>cpu_C;
    cpu_C<span style="color: #333333;">=</span>new <span style="color: #333399; font-weight: bold;">float</span>[SIZE];

    <span style="color: #888888;">// This is the familiar way of doing matrix multiplication in a serial</span>
    <span style="color: #888888;">// manner on CPU.</span>
    <span style="color: #333399; font-weight: bold;">float</span> sum;
    <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> row<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; row<span style="color: #333333;">&lt;</span>N; row<span style="color: #333333;">++</span>){
        <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> col<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; col<span style="color: #333333;">&lt;</span>N; col<span style="color: #333333;">++</span>){
            sum <span style="color: #333333;">=</span> <span style="color: #6600ee; font-weight: bold;">0.f</span>;
            <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> n<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; n<span style="color: #333333;">&lt;</span>N; n<span style="color: #333333;">++</span>){
                sum <span style="color: #333333;">+=</span> h_A[row<span style="color: #333333;">*</span>N<span style="color: #333333;">+</span>n]<span style="color: #333333;">*</span>h_B[n<span style="color: #333333;">*</span>N<span style="color: #333333;">+</span>col];
            }
            cpu_C[row<span style="color: #333333;">*</span>N<span style="color: #333333;">+</span>col] <span style="color: #333333;">=</span> sum;
        }
    }

    <span style="color: #888888;">// Compare GPU and CPU calculation results, for the same job.</span>
    <span style="color: #333399; font-weight: bold;">double</span> err <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">0</span>;
    <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> ROW<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; ROW <span style="color: #333333;">&lt;</span> N; ROW<span style="color: #333333;">++</span>){
        <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> COL<span style="color: #333333;">=</span><span style="color: #0000dd; font-weight: bold;">0</span>; COL <span style="color: #333333;">&lt;</span> N; COL<span style="color: #333333;">++</span>){
            err <span style="color: #333333;">+=</span> cpu_C[ROW <span style="color: #333333;">*</span> N <span style="color: #333333;">+</span> COL] <span style="color: #333333;">-</span> h_C[ROW <span style="color: #333333;">*</span> N <span style="color: #333333;">+</span> COL];
        }
    }

    cout <span style="color: #333333;">&lt;&lt;</span> <span style="background-color: #fff0f0;">"Error: "</span> <span style="color: #333333;">&lt;&lt;</span> err <span style="color: #333333;">&lt;&lt;</span> endl;

    <span style="color: #008800; font-weight: bold;">return</span> <span style="color: #0000dd; font-weight: bold;">0</span>;
}
</pre></td></tr></tbody></table></div>

<br />

`dev_array.h`

<div align="right">
<button onclick="javascript:copytoclipboard('csp2')" style="border: none">Copy snippet to clipboard!</button>
</div>
<div style="background: rgb(255, 255, 255); border: solid gray; height: 1000px; overflow: auto; padding: 0.0em 0.0em; width: auto;"><table><tbody><tr><td><pre style="color: grey; line-height: 125%; margin-bottom: 0px; margin-right: 5px; margin-top: 0px;">  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
 21
 22
 23
 24
 25
 26
 27
 28
 29
 30
 31
 32
 33
 34
 35
 36
 37
 38
 39
 40
 41
 42
 43
 44
 45
 46
 47
 48
 49
 50
 51
 52
 53
 54
 55
 56
 57
 58
 59
 60
 61
 62
 63
 64
 65
 66
 67
 68
 69
 70
 71
 72
 73
 74
 75
 76
 77
 78
 79
 80
 81
 82
 83
 84
 85
 86
 87
 88
 89
 90
 91
 92
 93
 94
 95
 96
 97
 98
 99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118</pre></td><td id="csp2"><pre style="line-height: 125%; margin: 0px;"><span style="color: #557799;">#ifndef _DEV_ARRAY_H_</span>
<span style="color: #557799;">#define _DEV_ARRAY_H_</span>

<span style="color: #557799;">#include &lt;stdexcept&gt;</span>
<span style="color: #557799;">#include &lt;algorithm&gt;</span>
<span style="color: #557799;">#include &lt;cuda_runtime.h&gt;</span>

<span style="color: #888888;">// Template of type - it's just like a place holder, specifying the declared</span>
<span style="color: #888888;">// variable is of SOME type (not known when being declared). Only when the </span>
<span style="color: #888888;">// declared variable is used do we know what its type is.</span>
<span style="color: #888888;">// See the following site for detailed explanation:</span>
<span style="color: #888888;">// http://www.cplusplus.com/doc/oldtutorial/templates/</span>
template <span style="color: #333333;">&lt;</span>class T<span style="color: #333333;">&gt;</span>

class dev_array
{
<span style="color: #997700; font-weight: bold;">public:</span>
    <span style="color: #888888;">// Refer to the following link for explanation about 'explicit':</span>
    <span style="color: #888888;">// https://stackoverflow.com/questions/121162/what-does-the-explicit-keyword-mean</span>
    <span style="color: #888888;">// </span>
    <span style="color: #888888;">// Here follows we can find explanation about the functionality of ":":</span>
    <span style="color: #888888;">// https://stackoverflow.com/questions/2785612/c-what-does-the-colon-after-a-constructor-mean</span>
    <span style="color: #888888;">// </span>
    <span style="color: #888888;">// Basicaly, we are initializing variables here.</span>
    explicit dev_array()
        <span style="color: #333333;">:</span> start_(<span style="color: #0000dd; font-weight: bold;">0</span>),
          end_(<span style="color: #0000dd; font-weight: bold;">0</span>)
    {}

    <span style="color: #888888;">// Refer to the following link for explanation about data type 'size_t':</span>
    <span style="color: #888888;">// https://en.cppreference.com/w/cpp/types/size_t</span>
    explicit dev_array(<span style="color: #333399; font-weight: bold;">size_t</span> size)
    {
        <span style="color: #888888;">// The 'allocate' function is responsible for allocating memory on </span>
        <span style="color: #888888;">// GPU - refer to the definition of 'allocate' function in the end of </span>
        <span style="color: #888888;">// current file - the private functions part.</span>
        allocate(size);
    }

    <span style="color: #333333;">~</span>dev_array()
    {
        free();
    }

    <span style="color: #888888;">// Both the variables 'start_' and 'end_', as we will see later in current</span>
    <span style="color: #888888;">// file, are pointers - 'start_' points to the starting position (in </span>
    <span style="color: #888888;">// memory) and 'end_' points to the end. Therefore, the function defined</span>
    <span style="color: #888888;">// here follows returns the size of the allocated array on GPU.</span>
    <span style="color: #888888;">// Refer to the following site for explanation about const member function</span>
    <span style="color: #888888;">// (i. e. the 'const' keyword after the function name below):</span>
    <span style="color: #888888;">// https://docs.microsoft.com/en-us/cpp/cpp/const-cpp?view=vs-2019</span>
    <span style="color: #333399; font-weight: bold;">size_t</span> getSize() <span style="color: #008800; font-weight: bold;">const</span>
    {
        <span style="color: #008800; font-weight: bold;">return</span> end_ <span style="color: #333333;">-</span> start_;
    }

    <span style="color: #888888;">// Return the starting position in memory, as a pointer.</span>
    T<span style="color: #333333;">*</span> getData() <span style="color: #008800; font-weight: bold;">const</span>
    {
        <span style="color: #008800; font-weight: bold;">return</span> start_;
    }

    <span style="color: #888888;">// Function responsible for copying data from host to device. 'start_'</span>
    <span style="color: #888888;">// specifies where the copied data will start on device; 'src' specifies</span>
    <span style="color: #888888;">// the source of the copying on host; 'sizeof(T)' specifies the unit of</span>
    <span style="color: #888888;">// the copied block and 'min' here gives how many such units of block will</span>
    <span style="color: #888888;">// be copied.  </span>
    <span style="color: #333399; font-weight: bold;">void</span> set(<span style="color: #008800; font-weight: bold;">const</span> T<span style="color: #333333;">*</span> src, <span style="color: #333399; font-weight: bold;">size_t</span> size)
    {
        <span style="color: #333399; font-weight: bold;">size_t</span> min <span style="color: #333333;">=</span> std<span style="color: #333333;">::</span>min(size, getSize());
        cudaError_t result <span style="color: #333333;">=</span> cudaMemcpy(start_, src, min <span style="color: #333333;">*</span> <span style="color: #008800; font-weight: bold;">sizeof</span>(T), cudaMemcpyHostToDevice);
        <span style="color: #008800; font-weight: bold;">if</span> (result <span style="color: #333333;">!=</span> cudaSuccess)
        {
            throw std<span style="color: #333333;">::</span>runtime_error(<span style="background-color: #fff0f0;">"failed to copy to device memory"</span>);
        }
    }

    <span style="color: #888888;">// Similar to the 'set' function above. In this case, we have the copy</span>
    <span style="color: #888888;">// direction reversed - from device to host.</span>
    <span style="color: #333399; font-weight: bold;">void</span> get(T<span style="color: #333333;">*</span> dest, <span style="color: #333399; font-weight: bold;">size_t</span> size)
    {
        <span style="color: #333399; font-weight: bold;">size_t</span> min <span style="color: #333333;">=</span> std<span style="color: #333333;">::</span>min(size, getSize());
        cudaError_t result <span style="color: #333333;">=</span> cudaMemcpy(dest, start_, min <span style="color: #333333;">*</span> <span style="color: #008800; font-weight: bold;">sizeof</span>(T), cudaMemcpyDeviceToHost);
        <span style="color: #008800; font-weight: bold;">if</span> (result <span style="color: #333333;">!=</span> cudaSuccess)
        {
            throw std<span style="color: #333333;">::</span>runtime_error(<span style="background-color: #fff0f0;">"failed to copy to host memory"</span>);
        }
    }

<span style="color: #997700; font-weight: bold;">private:</span>
    <span style="color: #888888;">// Allocate memory on device. The starting position in memory on device will</span>
    <span style="color: #888888;">// be returned as a pointer.</span>
    <span style="color: #333399; font-weight: bold;">void</span> allocate(<span style="color: #333399; font-weight: bold;">size_t</span> size)
    {
        cudaError_t result <span style="color: #333333;">=</span> cudaMalloc((<span style="color: #333399; font-weight: bold;">void</span><span style="color: #333333;">**</span>)<span style="color: #333333;">&amp;</span>start_, size <span style="color: #333333;">*</span> <span style="color: #008800; font-weight: bold;">sizeof</span>(T));
        <span style="color: #008800; font-weight: bold;">if</span> (result <span style="color: #333333;">!=</span> cudaSuccess)
        {
            start_ <span style="color: #333333;">=</span> end_ <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">0</span>;
            throw std<span style="color: #333333;">::</span>runtime_error(<span style="background-color: #fff0f0;">"failed to allocate device memory"</span>);
        }
        end_ <span style="color: #333333;">=</span> start_ <span style="color: #333333;">+</span> size;
    }

    <span style="color: #888888;">// Free memory on device.</span>
    <span style="color: #333399; font-weight: bold;">void</span> free()
    {
        <span style="color: #008800; font-weight: bold;">if</span> (start_ <span style="color: #333333;">!=</span> <span style="color: #0000dd; font-weight: bold;">0</span>)
        {
            cudaFree(start_);
            start_ <span style="color: #333333;">=</span> end_ <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">0</span>;
        }
    }

    T<span style="color: #333333;">*</span> start_;
    T<span style="color: #333333;">*</span> end_;
};

<span style="color: #557799;">#endif</span>
</pre></td></tr></tbody></table></div>

<br />

`kernel.h`

<div align="right">
<button onclick="javascript:copytoclipboard('csp3')" style="border: none">Copy snippet to clipboard!</button>
</div>
<div style="background: rgb(255, 255, 255); border: solid gray; overflow: auto; padding: 0.0em 0.0em; width: auto;"><table><tbody><tr><td><pre style="color: grey; line-height: 125%; margin-bottom: 0px; margin-right: 5px; margin-top: 0px;">1
2
3
4
5
6</pre></td><td id="csp3"><pre style="line-height: 125%; margin: 0px;"><span style="color: #557799;">#ifndef KERNEL_CUH_</span>
<span style="color: #557799;">#define KERNEL_CUH_</span>

<span style="color: #333399; font-weight: bold;">void</span> <span style="color: #0066bb; font-weight: bold;">matrixMultiplication</span>(<span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>A, <span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>B, <span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>C, <span style="color: #333399; font-weight: bold;">int</span> N);

<span style="color: #557799;">#endif</span>
</pre></td></tr></tbody></table></div>

<br />

`kernel.cu`

<div align="right">
<button onclick="javascript:copytoclipboard('csp4')" style="border: none">Copy snippet to clipboard!</button>
</div>
<div style="background: rgb(255, 255, 255); border: solid gray; height: 500px; overflow: auto; padding: 0.0em 0.0em; width: auto;"><table><tbody><tr><td><pre style="color: grey; line-height: 125%; margin-bottom: 0px; margin-right: 5px; margin-top: 0px;"> 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53</pre></td><td id="csp4"><pre style="line-height: 125%; margin: 0px;"><span style="color: #557799;">#include &lt;math.h&gt;</span>
<span style="color: #557799;">#include &lt;iostream&gt;</span>
<span style="color: #557799;">#include "cuda_runtime.h"</span>
<span style="color: #557799;">#include "kernel.h"</span>
<span style="color: #557799;">#include &lt;stdlib.h&gt;</span>

using namespace std;

<span style="color: #888888;">// This is where the real magic is happening - the matrix multiplication on GPU</span>
<span style="color: #888888;">// in a parallel manner.</span>
<span style="color: #008800; font-weight: bold;">__global__</span> <span style="color: #333399; font-weight: bold;">void</span> <span style="color: #0066bb; font-weight: bold;">matrixMultiplicationKernel</span>(<span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">*</span> A, <span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">*</span> B, <span style="color: #333399; font-weight: bold;">float</span><span style="color: #333333;">*</span> C, <span style="color: #333399; font-weight: bold;">int</span> N)
{
    <span style="color: #888888;">// Here, 'workers' are assigned task, according to their unique ID.</span>
    <span style="color: #888888;">// N. B. the way used here to assign rows and columns for 'workers'</span>
    <span style="color: #888888;">// guarantee that no two 'workers' will be assigned the same row and column</span>
    <span style="color: #888888;">// at the same time.</span>
    <span style="color: #333399; font-weight: bold;">int</span> ROW <span style="color: #333333;">=</span> <span style="color: #007020;">blockIdx</span>.y<span style="color: #333333;">*</span><span style="color: #007020;">blockDim</span>.y<span style="color: #333333;">+</span><span style="color: #007020;">threadIdx</span>.y;
    <span style="color: #333399; font-weight: bold;">int</span> COL <span style="color: #333333;">=</span> <span style="color: #007020;">blockIdx</span>.x<span style="color: #333333;">*</span><span style="color: #007020;">blockDim</span>.x<span style="color: #333333;">+</span><span style="color: #007020;">threadIdx</span>.x;

    <span style="color: #333399; font-weight: bold;">float</span> tmpSum <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">0</span>;

    <span style="color: #888888;">// Now each 'worker' knows exactly what he/she is responsible, it's time</span>
    <span style="color: #888888;">// for them to do their jobs, individually.</span>
    <span style="color: #008800; font-weight: bold;">if</span> (ROW <span style="color: #333333;">&lt;</span> N <span style="color: #333333;">&amp;&amp;</span> COL <span style="color: #333333;">&lt;</span> N) {
        <span style="color: #008800; font-weight: bold;">for</span> (<span style="color: #333399; font-weight: bold;">int</span> i <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">0</span>; i <span style="color: #333333;">&lt;</span> N; i<span style="color: #333333;">++</span>) {
            tmpSum <span style="color: #333333;">+=</span> A[ROW <span style="color: #333333;">*</span> N <span style="color: #333333;">+</span> i] <span style="color: #333333;">*</span> B[i <span style="color: #333333;">*</span> N <span style="color: #333333;">+</span> COL];
        }
    }
    C[ROW <span style="color: #333333;">*</span> N <span style="color: #333333;">+</span> COL] <span style="color: #333333;">=</span> tmpSum;
}


<span style="color: #333399; font-weight: bold;">void</span> <span style="color: #0066bb; font-weight: bold;">matrixMultiplication</span>(<span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>A, <span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>B, <span style="color: #333399; font-weight: bold;">float</span> <span style="color: #333333;">*</span>C, <span style="color: #333399; font-weight: bold;">int</span> N)
{
    <span style="color: #888888;">// Here, we use 'dim3' type variables for declaring the number of blocks </span>
    <span style="color: #888888;">// per grid and the number of threads per block.</span>
    <span style="color: #888888;">// Refer to the following link about 'dim3' in CUDA:</span>
    <span style="color: #888888;">// https://codeyarns.github.io/tech/2011-02-16-cuda-dim3.html</span>
    <span style="color: #888888;">// or </span>
    <span style="color: #888888;">// https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html</span>
    <span style="color: #333399; font-weight: bold;">dim3</span> threadsPerBlock(N, N);
    <span style="color: #333399; font-weight: bold;">dim3</span> blocksPerGrid(<span style="color: #0000dd; font-weight: bold;">1</span>, <span style="color: #0000dd; font-weight: bold;">1</span>);

    <span style="color: #888888;">// The IF statement guarantees that we have at most 512 threads per block.</span>
    <span style="color: #008800; font-weight: bold;">if</span> (N<span style="color: #333333;">*</span>N <span style="color: #333333;">&gt;</span> <span style="color: #0000dd; font-weight: bold;">512</span>){
        threadsPerBlock.x <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">32</span>;
        threadsPerBlock.y <span style="color: #333333;">=</span> <span style="color: #0000dd; font-weight: bold;">16</span>;
        blocksPerGrid.x <span style="color: #333333;">=</span> ceil(<span style="color: #333399; font-weight: bold;">double</span>(N)<span style="color: #333333;">/</span><span style="color: #0000dd; font-weight: bold;">32</span>);
        blocksPerGrid.y <span style="color: #333333;">=</span> ceil(<span style="color: #333399; font-weight: bold;">double</span>(N)<span style="color: #333333;">/</span><span style="color: #0000dd; font-weight: bold;">16</span>);
    }

    matrixMultiplicationKernel<span style="color: #333333;">&lt;&lt;&lt;</span>blocksPerGrid,threadsPerBlock<span style="color: #333333;">&gt;&gt;&gt;</span>(A, B, C, N);
}
</pre></td></tr></tbody></table></div>

<br />

<p align='center'>
========================I AM A SEPARATOR========================
</p>

<br />

<p style='text-align: justify'>
To compile the codes given above, one may want to first save each individual snippet to a separate file (with file name as exactly given on top of each snippet) and also guarantee that all files stay in the same folder. Then one can use the makefile given below to compile the codes to get the executable, by simply executing 'make' on terminal.
</p>

<br />

`Makefile`

<div align="right">
<button onclick="javascript:copytoclipboard('csp5')" style="border: none">Copy snippet to clipboard!</button>
</div>
<div style="background: rgb(255, 255, 255); border: solid gray; overflow: auto; padding: 0.em 0.0em; width: auto;"><table><tbody><tr><td><pre style="color: grey; line-height: 125%; margin-bottom: 0px; margin-right: 5px; margin-top: 0px;"> 1
 2
 3
 4
 5
 6
 7
 8
 9
10
11
12</pre></td><td id="csp5"><pre style="line-height: 125%; margin: 0px;"><span style="color: #996633;">NCC</span><span style="color: #333333;">=</span>nvcc

<span style="color: #996633;">OBJ</span><span style="color: #333333;">=</span>kernel.o main_mat_mul.o

<span style="color: #0066bb; font-weight: bold;">mat_mul</span><span style="color: #333333;">:</span> <span style="color: #6600ee; font-weight: bold;">$(OBJ)</span>
<span style="color: #008800; font-weight: bold;"><span>&nbsp;&nbsp; &nbsp;</span>$(</span>NCC<span style="color: #008800; font-weight: bold;">)</span> -o <span style="color: #996633;">$@</span> <span style="color: #996633;">$^</span>

<span style="color: #0066bb; font-weight: bold;">main_mat_mul.o</span><span style="color: #333333;">:</span> <span style="color: #6600ee; font-weight: bold;">kernel.o</span>
<span style="color: #008800; font-weight: bold;"><span>&nbsp;&nbsp; &nbsp;</span>$(</span>NCC<span style="color: #008800; font-weight: bold;">)</span> -c main_mat_mul.cu

<span style="color: #0066bb; font-weight: bold;">kernel.o</span><span style="color: #333333;">:</span>
<span style="color: #008800; font-weight: bold;"><span>&nbsp;&nbsp; &nbsp;</span>$(</span>NCC<span style="color: #008800; font-weight: bold;">)</span> -c kernel.cu
</pre></td></tr></tbody></table></div>

<br />

<b>References</b>

[1] [https://www.quantstart.com/articles/Matrix-Matrix-Multiplication-on-the-GPU-with-Nvidia-CUDA/](https://www.quantstart.com/articles/Matrix-Matrix-Multiplication-on-the-GPU-with-Nvidia-CUDA/)