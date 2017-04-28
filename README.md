jpg checker

* 상용 툴 사용이 아닌 직접 구현을 통해 jpg structure 검증 관련 내용입니다.
* **JFIF Segment 검증 외 Encoding/Decoding 관련 검증 구현은 지양하였습니다.**  

***

JPEG 이미지 파일은 JFIF(JPEG File Interchange Format) 파일 구조를 가집니다.

JFIF는 여러 Segment(Marker)의 나열로 구성되어 있습니다.

![](http://cfile26.uf.tistory.com/image/14091A1B4ACF482B6C8228)

![](https://ih1.redbubble.net/image.34570367.6973/flat,800x800,075,f.u1.jpg)

![](http://old.honeynet.org/scans/scan26/sol/claus/figure4.jpg)

원래는 다음과 같이 다양한 종류의 Segment가 있습니다만,

| Name | Code (hex) | Purpose |
| :---: | :---: | :---: |
| SOI | FFD8 | Start Of Image |
| APP0 | FFE0 | JFIF APP0 segment marker, |
| APP15 | FFEF | ignore |
| SOF0 | FFC0 | Start Of Frame (baseline JPEG), for details see below |
| SOF1 | FFC1 | Start Of Frame (baseline JPEG), for details see below |
| SOF2 | FFC2 | usually unsupported |
| SOF3 | FFC3 | usually unsupported |
| SOF5 | FFC5 | usually unsupported |
| SOF6 | FFC6 | usually unsupported |
| SOF7 | FFC7 | usually unsupported |
| SOF9 | FFC9 | for arithmetic coding, usually unsupported |
| SOF10 | FFCA | usually unsupported |
| SOF11 | FFCB | usually unsupported |
| SOF13 | FFCD | usually unsupported |
| SOF14 | FFCE | usually unsupported |
| SOF15 | FFCF | usually unsupported |
| DHT | FFC4 | Define Huffman Table |
| DQT | FFDB | Define Quantization Table |
| SOS | FFDA | Start Of Scan |
| JPG | FFC8 | undefined/reserved (causes decoding error) |
| JPG0 | FFF0 | ignore (skip) |
| JPG13 | FFFD | ignore (skip) |
| DAC | FFCC | Define Arithmetic Table, usually unsupported |
| DNL | FFDC | usually unsupported, ignore |
| DRI | FFDD | Define Restart Interval, for details see below |
| DHP | FFDE | ignore (skip) |
| EXP | FFDF | ignore (skip) |
| *RST0 | FFD0 | RSTn are used for resync, may be ignored |
| *RST1 | FFD1 | RSTn are used for resync, may be ignored |
| *RST2 | FFD2 | RSTn are used for resync, may be ignored |
| *RST3 | FFD3 | RSTn are used for resync, may be ignored |
| *RST4 | FFD4 | RSTn are used for resync, may be ignored |
| *RST5 | FFD5 | RSTn are used for resync, may be ignored |
| *RST6 | FFD6 | RSTn are used for resync, may be ignored |
| *RST7 | FFD7 | RSTn are used for resync, may be ignored |
| *TEM | FF01 | usually causes a decoding error, may be ignored |
| COM | FFFE | Comment, may be ignored   |
| EOI | FFD9 | End Of Image |


필수적으로 존재해야하는 Segment 종류는 다음과 같습니다.  
(원내의 jpg 파일들과 은행에서 전송받은 jpg 파일들로 검증 완)

| Name | Code (hex) | Purpose |
| :---: | :---: | :---: |
| SOI | FFD8 | Start Of Image |
| DQT | FFDB | One or more quantization tables DQT |
| SOF | FFC0 | Start Of Frame |
| DHT | FFC4 | One or more huffman tables DHT |
| SOS | FFDA | Start Of Scan |
| EOI | FFD9 | End of Image |

***

구현 이슈

* SOF Segment에서 내부적으로 quantization table을 참조하고
* SOS Segment에서 내부적으로 huffman table을 참조하기 때문에

Quantization table을 보관하고 있는 DQT Segment와  
Huffman table을 보관하고 있는 DHT Segment를 먼저 읽도록 구현하였습니다.

***

DQT Segment는 다음과 같이 구성되어 있습니다.
* 2 bytes : FFDB
* 2 bytes : 추가 데이터 길이(DQT Segment에서 Marker Identifier 제외한 길이)
* Quantization table N개(1 <= N <= 4)

Quantization table은 다음과 같이 구성됩니다.
* 1 byte : 상위 4 bit는 quantization table의 precision 값입니다.  
하위 4 bit는 quantization table의 id로 0~3  사이의 값입니다.
* ((precision + 1) * 64) bytes : quantization table

***

DHT Segment는 다음과 같이 구성되어 있습니다.
* 2 bytes : FFC4
* 2 bytes : 추가 데이터 길이(DHT Segment에서 Marker Identifier 제외한 길이)
* Huffman table N개(1 <= N <= 4)

Huffman table은 다음과 같이 구성됩니다.
* 1 byte : 상위 4 bit는 huffman table type을 나타냅니다.  
0은 DC, 1은 AC table을 의미합니다.  
하위 4 bit는 huffman table의 id로 0~3  사이의 값입니다.
* 16 bytes : 코드 길이(1 byte) 16개를 보관하며, 이 값의 합은 256보다 작거나 같아야 합니다.
* (코드 길이의 합) bytes : huffman code

***

SOF Segment는 그림 이미지의 크기 및 샘플링에 관한 정보를 보관하고 있으며 다음과 같이 구성되어 있습니다.
* 2 bytes : FFC0
* 2 bytes : 추가 데이터 길이(SOF Segment에서 Marker Identifier 제외한 길이)
* 1 byte : Data precision으로 8, 12 또는 16
* 2 bytes : Image height > 0
* 2 bytes : Image Width > 0
* 1 byte : Number of components으로 1(grey scaled), 3(color) 또는 4(color)
* Component specific N개(1 <= N <= 3)

Component specific은 다음과 같이 구성됩니다.
* 1 byte : Component id
* 1 byte : 샘플링 주기
* 1 byte : quantization table id  
-> **_앞서 DQT Segment에서 읽은 quantization table에 해당 id가 있는지 확인_**

***

SOS Segment는
* 각 성분들이 어떤 huffman table을 사용할지 알려주며
* SOS Segment가 시작되기 전에 최소한 하나 이상의 DHT Segment, DQT Segment와 하나의 SOF Segment가 있어야 합니다.
* SOS Segment 바로 다음에 이미지 스캔 데이터 블록이 오게 됩니다.

SOS Segment는 다음과 같이 구성되어 있습니다.
* 2 bytes : FFDA
* 2 bytes : 추가 데이터 길이(SOS Segment에서 Marker Identifier 제외한 길이)
* 1 byte : Scan description 개수 N(1 <= N <= 4)
* Scan description N개
* 3 bytes : 무시

Scan description은 다음과 같이 구성됩니다.
* 1 byte : scan description id
* 1 byte : 상위 4 bit는 DC huffman table id  
-> **_앞서 DHT Segment에서 읽은 huffman table(DC)에 해당 id가 있는지 확인_**  
하위 4 bit는 AC huffman table id  
-> **_앞서 DHT Segment에서 읽은 huffman table(AC)에 해당 id가 있는지 확인_**

***

참고 및 출처
* [JPEG File Layout and Format](http://vip.sugovica.hu/Sardi/kepnezo/JPEG%20File%20Layout%20and%20Format.htm)
* [[JPEG] JPEG의 파일구조](http://cometkorea.tistory.com/56)
* [jpeg file interchange format image(google 검색)](https://ih1.redbubble.net/image.34570367.6973/flat,800x800,075,f.u1.jpg)
* [jpeg file interchange format image(google 검색)](http://old.honeynet.org/scans/scan26/sol/claus/figure4.jpg)
* [Common JPEG markers](http://www.digicamsoft.com/itu/itu-t81-36.html)
* [JPEG - Video Technology Magazine](http://www.videotechnology.com/jpeg/j1.html)
* [File format structure](https://en.wikipedia.org/wiki/JPEG_File_Interchange_Format)
* [The Metadata in JPEG files](http://dev.exiv2.org/projects/exiv2/wiki/The_Metadata_in_JPEG_files)
* [JPEG - Idea and Practice/The header part](https://en.wikibooks.org/wiki/JPEG_-_Idea_and_Practice/The_header_part)
