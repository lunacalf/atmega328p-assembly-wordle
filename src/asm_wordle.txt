;---------------
; Assembly Code Wordle Game
;---------------
.include "m328pbdef.inc"

.cseg          ; กำหนดให้เป็น Code Segment
.org 0x0000    ; เริ่มต้นที่ตำแหน่ง 0x00
JMP main

.org 0x0002	   ; Vector สำหรับ INT0
JMP ISR_INT0

.org 0x0004    ; Vector สำหรับ INT1
JMP ISR_INT1

;------------------------
main:
    RCALL init     ; เรียกฟังก์ชันตั้งค่า IO
main_loop:
	RCALL read_DIP
	RCALL display_7seg ; แสดงผลบน 4-digit 7-segment
	RCALL display_led
    RJMP  main_loop    ; วนลูปไปเรื่อยๆ

;------------------------------------------------------------------
init:
	CLR   R2
	CLR   R3
	CLR   R4
	CLR   R5
	CLR   R19
	CLR   R20
	LDI   R20, 0x00   ; กำหนด PORTC เป็น input (PC0-PC3)
    OUT   DDRC, R20

    LDI   R20, 0xF3   ; PORTD 0-1,4-7 เป็น output (7-segmentและLED ใช้ shift-register )
    OUT   DDRD, R20

	LDI   R20, 0x0F   ; PORTB เป็น output (Digit Select)
    OUT   DDRB, R20

	LDI	  R20, 0x0F   ; กำหนด interrupt ให้ int0 int1 ทำงานที่ขอบขาขึ้น
	STS   EICRA, R20

	LDI	  R20, 0x03   ;	เปิดใช้งาน int0 int1
	OUT   EIMSK, R20

	LDI   R18, 0     ; เลือกตำแหน่งคำศัพท์ที่เป็นเฉลย
	LDI	  R19, 4     ; บันทึกค่าตำแหน่งที่กำลังใส่ input

	RCALL load_word
	SEI

    RET

;------------------------------------------------------------------
load_word:
	CPI   R18, 36          ; 36 คือขนาดของคลังคำศัพท์ (9 คำ * 4 ไบต์)
    BRLO  load_word_ok     ; ถ้า R18 < 36 ให้โหลดคำต่อไป
    CLR   R18              ; ถ้า R18 >= 36 ให้รีเซ็ตเป็น 0
load_word_ok:
    LDI   ZL, LOW(word * 2)  ; โหลดตำแหน่งเริ่มต้นของคลังคำศัพท์
    LDI   ZH, HIGH(word * 2)
    ADD   ZL, R18                 ; R18 เก็บค่าดัชนีของคำที่ต้องการ (เช่น 0 สำหรับคำแรก, 4 สำหรับคำที่สอง)
    ADC   ZH, R1
    LPM   R26, Z+                 ; โหลดตัวอักษรแรก
    LPM   R27, Z+                 ; โหลดตัวอักษรที่สอง
    LPM   R28, Z+                 ; โหลดตัวอักษรที่สาม
    LPM   R29, Z+                 ; โหลดตัวอักษรที่สี่
	
	; เพิ่มค่า R18 ไป 4 เพื่อชี้ไปยังคำถัดไป
    LDI   R22, 4
    ADD   R18, R22
    RET

;------------------------------------------------------------------
compare_words:
    CLR   R24          ; ล้างค่า R24 เพื่อเก็บผลลัพธ์การเปรียบเทียบ (หลักที่ 1-4)
	MOV   R7, R26	   ; บันทึกค่าไว้เพื่อใช้เปรียบเทียบ
	MOV	  R8, R27
	MOV   R9, R28
	MOV   R10, R29

check_pos1:
    ; เปรียบเทียบหลักที่ 1 (R2 กับ R26)
    CP    R2, R7
    BREQ  correct_pos1 ; ถ้าตรงกัน (ถูกต้องและตำแหน่งถูกต้อง)
    RJMP  check_pos2
correct_pos1:
    SBR   R24, 0b10000000 ; ตั้งค่า bit 7 ของ R24 เป็น 1 (LED เขียว)
	CLR   R7		  ; R7 ไม่ถูกใช้เพื่อเปรียบเทียบอีก
    RJMP  check_pos2

check_pos2:
    ; เปรียบเทียบหลักที่ 2 (R3 กับ R27)
    CP    R3, R8
    BREQ  correct_pos2
    RJMP  check_pos3
correct_pos2:
    SBR   R24, 0b01000000 ; ตั้งค่า bit 6 ของ R24 เป็น 1 (LED เขียว)
	CLR   R8		  ; R8 ไม่ถูกใช้เพื่อเปรียบเทียบอีก
	RJMP  check_pos3

check_pos3:
    ; เปรียบเทียบหลักที่ 3 (R4 กับ R28)
    CP    R4, R9
    BREQ  correct_pos3
    RJMP  check_pos4
correct_pos3:
    SBR   R24, 0b00100000 ; ตั้งค่า bit 5 ของ R24 เป็น 1 (LED เขียว)
	CLR   R9		      ; R9 ไม่ถูกใช้เพื่อเปรียบเทียบอีก
    RJMP  check_pos4

check_pos4:
    ; เปรียบเทียบหลักที่ 4 (R5 กับ R29)
    CP    R5, R10
    BREQ  correct_pos4
    RJMP  check_exist1
correct_pos4:
    SBR   R24, 0b00010000 ; ตั้งค่า bit 4 ของ R24 เป็น 1 (LED เขียว)
	CLR   R10		  ; R10 ไม่ถูกใช้เพื่อเปรียบเทียบอีก
    RJMP  check_exist1

check_exist1:
    ; ตรวจสอบว่าตัวอักษรอยู่ในเฉลยหรือไม่
	SBRC  R24, 7		  ; หากตำแหน่งนี้ถูกแล้วให้ข้ามไปหลักถัดไป
	RjMP  check_exist2
    CP    R2, R8
    BREQ  correct_char1_2
    CP    R2, R9
    BREQ  correct_char1_3
    CP    R2, R10
    BREQ  correct_char1_4
    RJMP  check_exist2
correct_char1_2:
    SBR   R24, 0b00001000 ; ตั้งค่า bit 3 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R8
	RJMP  check_exist2
correct_char1_3:
	SBR   R24, 0b00001000 ; ตั้งค่า bit 3 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R9
	RJMP  check_exist2
correct_char1_4:
	SBR   R24, 0b00001000 ; ตั้งค่า bit 3 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R10
	RJMP  check_exist2

check_exist2:
	SBRC  R24, 6		  ; หากตำแหน่งนี้ถูกแล้วให้ข้ามไปหลักถัดไป
	RjMP  check_exist3
    CP    R3, R7
    BREQ  correct_char2_1
    CP    R3, R9
    BREQ  correct_char2_3    
    CP    R3, R10
    BREQ  correct_char2_4
    RJMP  check_exist3
correct_char2_1:
    SBR   R24, 0b00000100 ; ตั้งค่า bit 2 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R7
	RJMP  check_exist3
correct_char2_3:
	SBR   R24, 0b00000100 ; ตั้งค่า bit 2 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R9
	RJMP  check_exist3
correct_char2_4:
	SBR   R24, 0b00000100 ; ตั้งค่า bit 2 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R10
	RJMP  check_exist3

check_exist3:
	SBRC  R24, 5		  ; หากตำแหน่งนี้ถูกแล้วให้ข้ามไปหลักถัดไป
	RjMP  check_exist4
    CP    R4, R7
    BREQ  correct_char3_1
    CP    R4, R8
    BREQ  correct_char3_2
    CP    R4, R10
    BREQ  correct_char3_4
    RJMP  check_exist4
correct_char3_1:
    SBR   R24, 0b00000010 ; ตั้งค่า bit 1 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R7
    RJMP  check_exist4
correct_char3_2:
    SBR   R24, 0b00000010 ; ตั้งค่า bit 1 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R8
    RJMP  check_exist4
correct_char3_4:
    SBR   R24, 0b00000010 ; ตั้งค่า bit 1 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R10
    RJMP  check_exist4

check_exist4:
	SBRC  R24, 4		  ; หากตำแหน่งนี้ถูกแล้วให้ข้ามไปหลักถัดไป
	RjMP  done
    CP    R5, R7
    BREQ  correct_char4_1
    CP    R5, R8
    BREQ  correct_char4_2
    CP    R5, R9
    BREQ  correct_char4_3
    RJMP  done
correct_char4_1:
    SBR   R24, 0b00000001 ; ตั้งค่า bit 0 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R7
    RJMP  done
correct_char4_2:
    SBR   R24, 0b00000001 ; ตั้งค่า bit 0 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R8
    RJMP  done
correct_char4_3:
    SBR   R24, 0b00000001 ; ตั้งค่า bit 0 ของ R24 เป็น 1 (LED เหลือง)
	CLR   R9
    RJMP  done

done:
    RET
;------------------------------------------------------------------
read_DIP:
	IN    R20, PINC   ; อ่านค่าจาก DIP Switch (PC0-PC4)
    ANDI  R20, 0x1F   ; ใช้แค่ 5-bit ล่างสุด (a-z)

    ; โหลดค่าจาก lookup table
    LDI   ZL, LOW(lookup_table * 2)
    LDI   ZH, HIGH(lookup_table * 2)
    ADD   ZL, R20
    ADC   ZH, R1
    LPM   R21, Z      ; โหลดค่าของ 7-segment pattern
    RET

;------------------------------------------------------------------
display_7seg:
    LDI   R25, 1     ; เริ่มต้นแสดงผลจากหลักที่ 1
    LDI   R20, 4     ; นับจำนวนหลัก (4 หลัก) เพื่อแสดงผล

display_loop:		 ; แสดงผล 4 ตำแหน่งแบบ multiplexing
	CBI   PORTD, 6	  ; ให้ LATCH เป็น low
	 ; ตรวจสอบตำแหน่งที่กำลังแก้ไข
    CP    R20, R19      ; เปรียบเทียบหลักที่กำลังแสดงผลกับหลักที่กำลังแก้ไข
    BREQ  show_current  ; ถ้าเป็นหลักที่กำลังแก้ไข ให้แสดงผลจาก R21

	SBRC  R25, 0
	MOV   R6, R2
	SBRC  R25, 1
	MOV   R6, R3
	SBRC  R25, 2
	MOV   R6, R4
	SBRC  R25, 3
	MOV   R6, R5
	RJMP  shift_out
show_current:
    MOV   R6, R21       ; แสดงผลจาก R21 (ค่าจาก DIP Switch)
shift_out:
	RCALL shiftout_7seg   ; ส่งข้อมูลไปยัง 7-segment
    OUT   PORTB, R25  ; เลือกหลักของ 7-segment
	SBI	  PORTD, 6	  ; ให้ LATCH เป็น high

    RCALL DELAY10MS   ; หน่วงเวลา 10 ms
    LSL   R25         ; ขยับไปหลักถัดไป (shift left)
    DEC   R20
    BRNE  display_loop

    RET
	
;------------------------------------------------------------------
display_led:
	CBI   PORTD, 1	  ; ให้ LATCH เป็น low
	MOV   R12, R24        ; สำรองค่า R24 ไว้ใน R12
	RCALL shiftout_led
	SBI	  PORTD, 1	  ; ให้ LATCH เป็น high
	RET
;------------------------------------------------------------------
shiftout_7seg:			  ; ใช้ shift register SIPO 
	PUSH  R16
	MOV   R11, R6         ; สำรองค่า R6 ไว้ใน R11
    LDI   R16, 7          ;counter for loop
nxt1:ROR   R11             ;put LSB of byte in C flag (LSB first)
    BRCS  output_1        ;jump to label if C = 1
    CBI   PORTD, 5        ;o/p logic 0
bak1:SBI   PORTD, 7        ;CLK high pulse
    CBI   PORTD, 7        ;CLK low pulse
    DEC   R16             ;decrement counter
    BRNE  nxt1
	POP   R16
    RET

output_1:
    SBI   PORTD, 5        ;o/p logic 1
    RJMP  bak1

;-------------------------------------------------------------------
shiftout_led:			  ; ใช้ shift register SIPO
	PUSH  R17
    LDI   R17, 8          ;counter for loop
nxt2:ROR   R12             ;put LSB of byte in C flag (LSB first)
    BRCS  output_2        ;jump to label if C = 1
    CBI   PORTD, 0        ;o/p logic 0
bak2:SBI   PORTD, 4        ;CLK high pulse
    CBI   PORTD, 4        ;CLK low pulse
    DEC   R17             ;decrement counter
    BRNE  nxt2
	POP   R17
    RET

output_2:
    SBI   PORTD, 0        ;o/p logic 1
    RJMP  bak2

;------------------------------------------------------------------
DELAY10MS:	push	R16				
		push	R17
		ldi	R16, 0x00
LOOP2:		inc	R16
		ldi	R17,  0x00
LOOP1:		inc	R17
		cpi	R17, 249
		brlo	LOOP1
		nop
		cpi	R16, 160
		brlo	LOOP2
		pop	R17
		pop	R16
		ret

;------------------------------------------------------------------
lookup_table:
	.DB 0b01110111, 0b01111100  ; A, B
    .DB 0b00111001, 0b01011110  ; C, D
    .DB 0b01111001, 0b01110001  ; E, F
    .DB 0b00111101, 0b01110100  ; G, H
    .DB 0b00000110, 0b00011110  ; I, J
    .DB 0b01110101, 0b00111000  ; K, L
    .DB 0b00010101, 0b01010100  ; M, N
    .DB 0b00111111, 0b01110011  ; O, P
    .DB 0b01100111, 0b00110001  ; Q, R
    .DB 0b01101101, 0b01111000  ; S, T
    .DB 0b00111110, 0b00011100  ; U, V
    .DB 0b00011101, 0b01110110  ; W, X
    .DB 0b01101110, 0b01011011  ; Y, Z

;------------------------------------------------------------------
word:
	.DB 0b01111000, 0b01111001, 0b01101101, 0b01111000  ; T, E, S, T
    .DB 0b01110011, 0b00111000, 0b01110111, 0b01101110  ; P, L, A, Y
    .DB 0b01110100, 0b00111111, 0b01110011, 0b01111001  ; H, O, P, E
    .DB 0b00111000, 0b00111111, 0b00011100, 0b01111001  ; L, O, V, E
    .DB 0b00111101, 0b01110111, 0b00010101, 0b01111001  ; G, A, M, E
    .DB 0b00011101, 0b00111111, 0b00110001, 0b01011110  ; W, O, R, D
    .DB 0b01100111, 0b00111110, 0b00000110, 0b01111000  ; Q, U, I, T
    .DB 0b00011110, 0b00111110, 0b00010101, 0b01110011  ; J, U, M, P
    .DB 0b01110110, 0b00110001, 0b01110111, 0b01101110  ; X, R, A, Y

;------------------------------------------------------------------
ISR_INT0:
	PUSH  R16         
    IN    R16, SREG  

	RCALL DELAY10MS    ; หน่วงเวลาเพื่อลด bouncing
	; ตรวจสอบสถานะปุ่มอีกครั้ง
    SBIC  PIND, 2       ; ตรวจสอบว่าปุ่ม INT0 ยังคงถูกกดอยู่หรือไม่
    RJMP  update_ISR0   ; ถ้าปุ่มถูกปล่อยแล้ว ให้ออกจาก ISR

	OUT   SREG, R16   
    POP   R16
	RETI                ; ออกจาก ISR

update_ISR0:
	CPI   R19, 4
	BREQ  CHOOSE1
	CPI   R19, 3
	BREQ  CHOOSE2
	CPI   R19, 2
	BREQ  CHOOSE3
	CPI   R19, 1
	BREQ  CHOOSE4
	CPI   R19, 0		; เมื่อเป็น 0 ให้วนไปดูหลักแรกใหม่
	BREQ  CHOOSE_LOOP

exit_ISR0:
	RCALL DELAY10MS
	SBIS  PIND, 2       ; ตรวจสอบว่าปุ่ม INT0 ยังคงถูกปล่อยหรือไม่
    RJMP  exit_ISR0     ; ถ้าปุ่มถูกกดอยู่ ให้วนเช็คใหม่
	RCALL DELAY_DEBOUNCE0 
    DEC   R19           ; หมุนเวียนหลักถัดไป
	OUT   SREG, R16   
    POP   R16

	RETI                ; ออกจาก ISR

CHOOSE1:
	MOV   R2, R21
    RJMP  exit_ISR0
CHOOSE2:
	MOV   R3, R21
    RJMP  exit_ISR0
CHOOSE3:
	MOV   R4, R21
    RJMP  exit_ISR0
CHOOSE4:
	MOV   R5, R21
    RJMP  exit_ISR0
CHOOSE_LOOP:		   ; เป็นตำแหน่งพัก ก่อนเลือกใหม่ ให้กดอีกครั้ง
	LDI   R19, 5
	RJMP  exit_ISR0
;------------------------------------------------------------------
ISR_INT1:
	PUSH  R16         
    IN    R16, SREG     

	RCALL DELAY10MS    ; หน่วงเวลาเพื่อลด bouncing
	; ตรวจสอบสถานะปุ่มอีกครั้ง
    SBIC  PIND, 3       ; ตรวจสอบว่าปุ่ม INT1 ยังคงถูกกดอยู่หรือไม่
    RJMP  update_ISR1   ; ถ้าปุ่มถูกปล่อยแล้ว ให้ออกจาก ISR

	OUT   SREG, R16   
    POP   R16
	RETI                ; ออกจาก ISR

update_ISR1:
	RCALL compare_words ; เรียกฟังก์ชันเปรียบเทียบ
    RCALL display_led ; แสดงผลบน LED
	CPI   R24, 0b11110000 ; ตรวจสอบว่า LED เขียวทั้ง 4 ดวง
	BREQ  next_word      ; ถ้าตอบถูกทั้งหมด ให้โหลดคำถัดไป
	RJMP  exit_ISR1     ; ออกจาก ISR
next_word:
    RCALL load_word     ; โหลดคำถัดไปจากคลังคำศัพท์

exit_ISR1:
    RCALL DELAY10MS     ; หน่วงเวลาเพิ่มเติม
	SBIS  PIND, 3       ; ตรวจสอบว่าปุ่ม INT1 ยังคงถูกปล่อยหรือไม่
    RJMP  exit_ISR1     ; ถ้าปุ่มถูกกดอยู่ ให้วนเช็คใหม่
	RCALL DELAY_DEBOUNCE1 
	    OUT   SREG, R16   
    POP   R16
    RETI   

;------------------------------------------------------------------
DELAY_DEBOUNCE0:
	PUSH  R22
    LDI   R22, 0xFF        
debounce_loop0:
    SBIC  PIND, 2       
    RJMP  debounce_loop0   
    
    DEC   R22               
    BRNE  debounce_loop0
	POP   R22
    RET      

;------------------------------------------------------------------
DELAY_DEBOUNCE1:
	PUSH  R22
    LDI   R22, 0xFF        
debounce_loop1:
    SBIC  PIND, 3       
    RJMP  debounce_loop1   
    
    DEC   R22               
    BRNE  debounce_loop1
	POP   R22
    RET  
;------------------------------------------------------------------
