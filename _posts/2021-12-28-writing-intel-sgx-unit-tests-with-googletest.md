---
layout: post
title: "Writing Intel SGX Unit Tests with GoogleTest"
---
No one likes writing unit tests. It's tedious and people usually do it only to achieve the next *milestone* in a project (sadly). However, unit tests come in handy when you develop a new algorithm and iteratively improve its performance. You want to make sure that with each iteration not only you improve the performance but also keep the correctness of your solution.
This working methodology remains the same for new hardware platforms, such as Intel SGX. However, with the peculiar way SGX programs compile it is non-trivial to integrate them with the existing C/C++ testing frameworks. I have not found any existing solution to this problem and decided to give it try. This post presents my approach to writing unit tests for Intel SGX applications using the GoogleTest framework. The remaining of the post explains step-by-step how I refactored *Sample Enclave*, one of the sample SGX projects provided by Intel, to work with GoogleTest.

## Build Tool

While GoogleTest supports Bazel and CMake, Intel SGX sample apps come with just Makefiles. However, there are already some SGX projects online that use CMake so there's no need to reinvent the wheel. I used [SGX-CMake](https://github.com/xzhangxa/SGX-CMake/) to rewrite the *Sample Enclave* project with CMake.
This part was rather straightforward. It required pasting *FindSGX.cmake* file and appending it in the project's CMakeLists:

```
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
find_package(SGX REQUIRED)
```

Next, I set the sources for the trusted part, added trusted libraries and signed the enclave:

```
set(EDL_SEARCH_PATHS Enclave)
set(E_SRCS Enclave/Enclave.cpp)
set(T_SRCS Enclave/TrustedLibrary/Libc.cpp Enclave/TrustedLibrary/Libcxx.cpp Enclave/TrustedLibrary/Thread.cpp
        Enclave/Edger8rSyntax/Arrays.cpp Enclave/Edger8rSyntax/Functions.cpp Enclave/Edger8rSyntax/Pointers.cpp Enclave/Edger8rSyntax/Types.cpp)
set(LDS Enclave/Enclave.lds)

add_trusted_library(trusted_lib SRCS ${T_SRCS} EDL Enclave/Enclave.edl EDL_SEARCH_PATHS ${EDL_SEARCH_PATHS})
add_enclave_library(enclave SRCS ${E_SRCS} TRUSTED_LIBS trusted_lib EDL Enclave/Enclave.edl EDL_SEARCH_PATHS ${EDL_SEARCH_PATHS} LDSCRIPT ${LDS})
enclave_sign(enclave KEY Enclave/Enclave_private_test.pem CONFIG Enclave/Enclave.config.xml)
```

Finally, I did the same for the untrusted part and built the untrusted executable:

```
set(SRCS App/App.cpp App/TrustedLibrary/Libc.cpp App/TrustedLibrary/Libcxx.cpp App/TrustedLibrary/Thread.cpp
        App/Edger8rSyntax/Arrays.cpp App/Edger8rSyntax/Functions.cpp App/Edger8rSyntax/Pointers.cpp App/Edger8rSyntax/Types.cpp)
add_untrusted_executable(app SRCS ${SRCS} EDL Enclave/Enclave.edl EDL_SEARCH_PATHS ${EDL_SEARCH_PATHS})
```

If we compile at this stage, we have the equivalent of the original project. The project outputs an *app* executable that runs a sample enclave.

## GoogleTest

The next step was to integrate GoogleTest into the project. This was trivial and required only adding a few lines to the CMake file:

```
include(FetchContent)
FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip
)
FetchContent_MakeAvailable(googletest)
enable_testing()
```

## Writing SGX Unit Tests

We can now proceed to testing parts of the trusted library. I started with the *Arrays* functions and created *arrays_test.cpp* file.
SGX tests verify the correctness of enclave calls (ECALLs). Therefore, it would be useful to have some sort of **Before** and **After** calls that create and destroy enclaves. This can be easily done using *Test* class provided by GoogleTest:

```
class ArraysTest : public ::testing::Test {
protected:
    void SetUp() override {
        sgx_status_t ret = SGX_ERROR_UNEXPECTED;
        ret = sgx_create_enclave(ENCLAVE_FILENAME, SGX_DEBUG_FLAG,NULL,NULL,&test_eid,NULL);
        if (ret != SGX_SUCCESS) {
            // print error
        }
    }

    void TearDown() override {
        sgx_destroy_enclave(test_eid);
    }

    sgx_enclave_id_t test_eid = 0;
};
```

Now that we have everything ready we can write our first test:

```
TEST_F(ArraysTest, ArrayUserCheck) {
    int arr[4] = {0, 1, 2, 3};
    sgx_status_t ret = SGX_ERROR_UNEXPECTED;
    ret = ecall_array_user_check(test_eid, arr);
    EXPECT_EQ(ret, SGX_SUCCESS);
    for (int i = 0; i < 4; i++) {
        EXPECT_EQ(arr[i], (3 - i));
    }
}
```

We follow with other tests by rewriting the *asserts* from the original application.

To compile the test we have to include it as an untrusted executable:

```
add_untrusted_executable(arrays_test
                         SRCS App/Edger8rSyntax/Functions.cpp App/Edger8rSyntax/Pointers.cpp test/arrays_test.cc
                         EDL Enclave/Enclave.edl
                         EDL_SEARCH_PATHS ${EDL_SEARCH_PATHS})
```

This step required a small refactoring. We can't include the *App.cpp* file in the test executable because of its main function. The main function messes up with GoogleTest which invokes any main present in its sources instead of the test class. After excluding *App.cpp* from the sources, I had to move the implementations of functions defined in *App.h* to *App.cpp* (e.g., edger8r\_pointer\_attributes).

Finally, we link the test file with GoogleTest:

```
target_link_libraries(
        arrays_test
        gtest_main
)
include(GoogleTest)
gtest_discover_tests(arrays_test)
```

We can now build the project:

```
mkdir build && cd build
cmake -S .. -B . && cmake --build .
```

And run the tests:

```
ctest
```

We should get results similar to:

```
Test project /home/kaichi/repo/SGX-GoogleTest/build
    Start 1: ArraysTest.ArrayUserCheck
1/4 Test #1: ArraysTest.ArrayUserCheck ........   Passed    0.05 sec
    Start 2: ArraysTest.ArrayIn
2/4 Test #2: ArraysTest.ArrayIn ...............   Passed    0.04 sec
    Start 3: ArraysTest.ArrayOut
3/4 Test #3: ArraysTest.ArrayOut ..............   Passed    0.03 sec
    Start 4: ArraysTest.Isary
4/4 Test #4: ArraysTest.Isary .................   Passed    0.03 sec

100% tests passed, 0 tests failed out of 4

Total Test time (real) =   0.15 sec
```

And that's it!

You can compile and run the tests yourself using the [source code](https://github.com/kai-chi/SGX-GoogleTest) available on Github.
