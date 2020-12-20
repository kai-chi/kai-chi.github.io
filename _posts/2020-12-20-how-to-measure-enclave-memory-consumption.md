---  
layout: post
title: "How to measure enclave's memory consumption on Intel SGX"
categories: sgx
---

1. Make sure you have `sgx-gdb` installed on your machine. Run: ```$ sgx-gdb --version ``` to check it.
2. Create a `.gdbinit` file in your project directory.
3. Paste this into the file:  
   ```
   # enable SGX memory measurement tool
   enable sgx_emmt

   # exit GDB after successfull execution
   set $_exitcode = -999
   define hook-stop
     if $_exitcode != -999
       quit
     end 
   end
   ```
4. In your global gdbinit file (in my case, `~/.gdbinit`) paste these lines:  
   ```
   set auto-load local-gdbinit on
   add-auto-load-safe-path /path/to/project/.gdbinit
   ```
5. In your application, make sure you are destroying the enclave using `sgx_destroy_enclave(const sgx_enclave_id_t enclave_id)`.
6. Run your SGX program using SGX-GDB:  
   ```
   $ sgx-gdb -ex=r --args your_executable arg1 arg2
   ```
   
   what it means:  
   `ex=r` - tells GDB to run the program immediately  
   `--args` - everything that follows it is a command and its args
   
   You could set an alias for this command to make it even smoother.
   
7. In the output you will now find the memory usage, e.g.,  
   ```
   [Peak stack used]: 14 KB
   [Peak heap used]:  67764 KB
   [Peak reserved memory used]:  0 KB
   ```
