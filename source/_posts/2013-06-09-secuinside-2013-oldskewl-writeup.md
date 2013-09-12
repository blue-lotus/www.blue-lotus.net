---
title: Secuinside 2013 oldskewl Writeup
author: fish
layout: post
permalink: /secuinside-2013-oldskewl-writeup/
categories:
  - CTF
  - writeup
---
The binary is obfuscated by adding random *0xF0* in order to stop IDA from analyzing the binary correctly. So first we should de-obfuscate the binary.

After de-obfuscation, we could find a tiny VM engine after spending some time in analysis. Function *0x4015b0* is the vital function for initializing VM dispatching functions. Sorry for those ambiguous function names <img src='http://www.blue-lotus.net/wp-includes/images/smilies/icon_razz.gif' alt=':P' class='wp-smiley' /> .

    /* 0x4015b0 */
    struct_vm *__thiscall init_struct_vm(struct_vm *this)
    {
      this->func1 = (int)func_acc_reg1;
      this->func2 = (int)func_read_data;
      this->func3 = (int)func_dereference;
      this->func4 = (int)func_shift_left;
      this->func5 = (int)func_nop_n;
      this->func6 = (int)func_nop_n_if_reg1_is_not_0;
      this->func7 = (int)func_end;
      return this;
    }
    

After that it&#8217;s trivial to rip the actual VM&#8217;ed code out and reverse it into human-understandable logics. The code appears to be reading three strings out from the sokcet and hashing them. At first we thought it to be a hash-construction problem, but a hint told us a dictionary is needed. Then we had to write a Python script to bruteforce for hash collisions.

    def calc(lst):
        x = 0
        for i in range(0, len(lst)):
            x = ((x << 16) & 0xffffffff) + ((x << 6) & 0xffffffff) + x + ord(lst[i : i + 1])
            #x = ord(lst[i : i + 1]) + x * 65601
            x = x & 0xffffffff
        return x
    
    def main():
        for x in range(0, 0xff + 1):
            data = 0
            for y in range(0, 0xff + 1):
                for z in range(0, 0xff + 1):
                    lst = []
                    lst.append(x)
                    lst.append(y)
                    lst.append(z)
                    data = calc(lst)
                    if data == 0x38B7E73C:
                        print "Done!"
            print hex(data)
    
    def find(base, index):
        if index >= 15:
            return -1
        for i in range(0, 2000):
            d = base + i * 0x100000000
            x = d % 65601
            if x <= 0x7f and x > 0x0a:
                new_base = d - x
                new_base = new_base / 65601
                if new_base <= 0x7f and new_base > 0x0a:
                    print "%02x %02x" % (new_base, x),
                    return 0
                if find(new_base, index + 1) != -1:
                    print "%02x" % x,
                    return 0
        return -1
    
    if __name__ == '__main__':
        remainder = 0
        base = []
        base.append(0x38b7e73c)
        base.append(0x991181ae)
        base.append(0x478692f9)
        # find(base, 0)
        f = open('wordsEn.TXT')
        data = f.read()
        f.close()
        strs = data.split('\n')
        for s in strs:
            result = calc(s)
            for b in base:
                if result == b:
                    print "%x %s" % (b, s)
    
        for s1 in strs:
            print s1
            for s2 in strs:
                s = s1 + s2
                result = calc(s)
                for b in base:
                    if result == b:
                        print "%x %s" % (b, s)
    
        print "finished"
    

Finally we got the flag: **codename ancien regime**