<ol>
  <li>
    <h1>文件管理的作用：</h1>
    <h3>个人感觉：</h3>
    <ol>
      <li>
        封装，提供接口，层次化
      </li>
      <li>
        提高存取效率，减轻管理压力
      </li>
    </ol>
    <p>主流的系统有NTFS、FAT、exFAT、EXT等，这里主要介绍FAT。</p>
  </li>
  <li>
    <h1>FAT文件系统简介：</h1>
    <p>FAT(short for File Allocation Table)文件系统是个通用文件系统，兼容主流OS，如Linux、Windows等，基础于简单的技术，是Wimdows的默认文件系统。由于过于简单的结构，FAT存在过度碎片化、文件损坏与文件名、大小限制等问题。</p>
    <p>FAT具有以下属性：</p>
    <ul>
      <li>
        分区不能超过2TB(WIndows无法将大于32G的光盘格式化为FAT32，但是Mac可以。)
      </li>
      <li>
        存储到FAT分区的文件不能超过4GB。
      </li>
      <li>
        FAT分区需要经常进行碎片整理以保持性能。
      </li>
      <li>
        通常不建议大于32G的FAT分区存在，因为这样的空间两不适配FAT过于简单的组织结构。
      </li>
    </ul>
    <p>FAT多用于小容量设备，其中操作系统之间的可移植性很重要，为硬盘选择文件系统时，不建议FAT，除非是旧版本Windows</p>
    <p><b style="color:red;">注意：</b>早期部分版本使用的是FAT16，他有很多技术问题以及使用限制，不建议在任何现在的媒体上使用。</p>
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
  <li>
    
  </li>
</ol>
