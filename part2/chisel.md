## scala basic

* var and val
* conditional returns a value
* method
    - overload
    - revursive and nest (scope)
* list
* for
* class
* code blocks
    - The last line of Scala code becomes the return value of the code block
    - A one-line code block doesn't need to be enclosed in {}
* List
    - map
    - zip
    - zipWithIndex
    - reduce
    - fold
    - scan
* Seq

### Types

* int
* double
* class java.lang.String
* Unit为返回值的Function不返回任何东西
* Boolean: 用于if
* asInstanceOf 强制转化
* Chisel Types
    - UInt
    - Bool: 用于when
    - asTypeOf asUInt asSInt


### class

* abstract classes
* traits
* objects = static class
* companion objects
* case classes

## Chisel

* Module
* Bundle
* Combinational Logic
    - Wire
* Control Flow
* Sequential Logic
    - Reg
    - RegNext
    - RegInit
    - withReset
    - withClock
    - withClockAndReset
    - 异步呢？


* Map
* map.get returns Option
    - None
    - Some
* Match/Case Statements
* IOs with Optional Fields
    - Optional IO with Option
    - Optional IO with Zero-Width Wires
* case class



* Vec
* Decoupled
* Queue
* arbiters
* Bitwise
    - popCount
    - reverse
* Onehot
    - UIntToOH
    - OHToUInt
* Muxes
    - PriorityMux
    - Mux1H
* Counter



### Tester

* poke
* peek
* step
* expect
* initiate