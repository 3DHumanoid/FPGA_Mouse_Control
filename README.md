# FPGA_Mouse_Control - Coded in Verilog
FPGA Mouse Control

ECEN 340 
Final Lab Project
Brian Allen
4/4/2022

Purposes:
1. To define a complex digital system
2. To collaborate with others on your design
3. To implement a complex digital system
4. To give a technical report


Technology Inclusion List:
Finite State Machine (at least 3 states)
Real – Time Memory access at 100MHz
Higher level Math Functions (Multiply or Divide)
Keyboard or Mouse I/O
7-segment display
External Sensors (light, sound, off-board switches)
VGA Port
Parallel Interface to Other Board
Serial Interface to Other Board
Analog to Digital Converters (ADCs) or Digital to Analog Converters (DACs)
Other approved technology

Description of Project:
	
	This project used the USB HID Host port on the Basys3 board. This chip has built in PIC24 drivers that can connect to a PS2 type mouse or keyboard. In this project I used this to connect to a mouse to receive data from the mouse. This project also utilizes the 7-segment display on the Basys3 board. 
When you click on the left mouse button and move the mouse around the number on the seven segment display increases. When you click the right mouse button and move the mouse around the number on the seven segment display decreases. Below is a  picture of how the mouse connection to the Basys3 board.
 
      (Mouse Connection to Basys 3)

The Basys 3 manual has further details on how the usb and PS2 mouse connect. The mouse connection is a little more complicated than the PS2 keyboard connection because the mouse is an Inout signal. Meaning that there can be data sent to the mouse as well as data received from the mouse. 
The mouse knows when to send and receive data through two wires the PS2 Clock and the PS2 Data. When the mouse is ready to send data it will drop the clock line low and then drop the data line low. After this it waits 100 microseconds and then raises the clock line. Once the clock line is raised then it sends 3 packets of 11 bits. 


Verilog Code:

Some code was used from ([FPGA tutorial] How to interface a mouse with Basys 3 FPGA - FPGA4student.com)

Control Top:

module control_top(
    input clk, btnC, 
    inout PS2Clk, PS2Data,
    output [6:0] seg,
    output [3:0] an,
    output [2:0] led,
    output dp 
    );
    
    wire clk_sseg, clk_mouse;
    //wire [3:0] digit_sel;
    wire [15:0] digit_display;
    wire [6:0] out_num;
    //wire [2:0] control;
    wire [7:0] mouse_data;
    wire [8:0] x_vel, y_vel, x_data, y_data;
    
    reg PS2CIO, PS2DIO;
    wire Clk_en, Data_en;
    
    clk_gen C1 (.clk(clk), .rst(btnC), .clk_sseg(clk_sseg), .clk_mouse(clk_mouse));
    digit_selector G1 (.clk(clk_sseg), .rst(btnC), .digit_sel(an));
    num_generator N1 (.digit_sel(an), .num(digit_display), .out_num(out_num));
    mouse_data M1 (.clk(clk), .mouse_clk(PS2Clk), .rst(btnC), .mouse_data(PS2Data), .Clk_en(Clk_en), .Data_en(Data_en),         .                                   .display_number(digit_display));
    sseg S1 (.num(out_num), .seg(seg));
     
endmodule

Clk_Gen:

module clk_gen(
    input clk,
    input rst,
    output clk_sseg, clk_mouse
    );
    
    reg [26:0] count;
    
    //initial count = 0;
    
    always @ (posedge clk, posedge rst)
        if(rst)
            count <= 0;
        else
            count <= (count + 1);
            
    assign clk_sseg = count[18]; //Clock for Display
    assign clk_mouse = count[2]; //Clock for mouse 100 MHz
    
endmodule


Digit Selector:

module digit_selector(
    input clk,
    input rst,
    output reg [3:0] digit_sel
    );
    reg [3:0] next_digit_sel;
    
    always @ (posedge clk, posedge rst)
        if(rst)
            digit_sel <= 4'b1110;
        else
            digit_sel <= next_digit_sel;
            
    always @ (digit_sel)
            case (digit_sel)
                4'b1110: next_digit_sel = 4'b1101;
                4'b1101: next_digit_sel = 4'b1011;
                4'b1011: next_digit_sel = 4'b0111;
                default: next_digit_sel = 4'b1110; 
            endcase    
                     
endmodule

Num_Generator:

module num_generator(
    input [3:0] digit_sel,
    input [15:0] num,
    output reg [3:0] out_num
    );
    
    always @ (digit_sel, num)
        case (digit_sel)
            4'b0111: out_num <= num/1000; //Number divided to be far left
            4'b1011: out_num <= (num % 1000)/100; //Number divided to be second from left
            4'b1101: out_num <= ((num % 1000) % 100)/10; //Number divided to be third from left
            4'b1110: out_num <= (((num % 1000) % 100) % 10); //Number divided to be far right
        endcase    
endmodule

Mouse Data:

module mouse_data(
    input clk, rst,
    inout mouse_data, mouse_clk, //Data is send both ways
    output Clk_en, Data_en,
    output reg [15:0] display_number
    //output [2:0] control_state
    );
    
    reg [10:0] shift1, shift2, shift3;
    reg [5:0] mouse_bits;
    reg counter, dCounter, negOld, negNew;

    //Need to put in Tristate Buffer
    
    always @(posedge mouse_clk or posedge rst)
    begin
        if(rst)
            mouse_bits <= 0;
        else if(mouse_bits <=31) 
            mouse_bits <= mouse_bits + 1;
        else 
             mouse_bits <= 0;
    
    end
        
    // Increase/Decrease the number when pressing Left/Right Mouse 
    always @(negedge mouse_clk or posedge rst)
    begin
        if(rst)
            display_number <= 0;
        else begin
            if(mouse_bits==1) begin
                if(mouse_data==1) // if The mouse is left clicked, increase the number 
                   display_number <= display_number + 1;
            end
            else if(mouse_bits==2) begin
               if(mouse_data==1&&display_number>0)// if The mouse is right clicked, decrease the number 
                   display_number <= display_number - 1;
                end
        end 
    end 
endmodule

Sseg:

module sseg(
    input [3:0] num,
    output reg [6:0] seg
    );
    always @(num)
    begin
        case(num)
        4'h0: seg = 7'b1000000;
        4'h1: seg = 7'b1001111;
        4'h2: seg = 7'b0100100;
        4'h3: seg = 7'b0110000;
        4'h4: seg = 7'b0011001;
        4'h5: seg = 7'b0010010;
        4'h6: seg = 7'b0000010;
        4'h7: seg = 7'b1111000;
        4'h8: seg = 7'b0000000;
        4'h9: seg = 7'b0010000;
//        4'hA: seg = 7'b0001000;
//        4'hB: seg = 7'b0000011;
//        4'hC: seg = 7'b1000110;
//        4'hD: seg = 7'b0100001;
//        4'hE: seg = 7'b0000110;
//        4'hF: seg = 7'b0001110;
    endcase
   end 
endmodule


Extra Information:

Worst Negative Slack (WNS): 7.722ns


Conclusion:
	
	This project was a little more difficult than I anticipated. The hardest part was figuring out how to get the timing right to initialize the mouse and tell it to start sending information. My original plan was to be able to manipulate the seven-segment display to display the different values that the mouse was sending. Including the x velocity, y velocity, and a count. I ended up just doing the count as I was unable to separate the information. 
	As of functionality although the program did not meet all the requirements it does function the way it is supposed to with the code that is there. Below is a video attachment showcasing the functionality of the project.

