module lcd_test(clk, reset2, start, RS, data, done, sel);
	input clk, reset2, done, sel;
	output start, RS;
	output [7:0] data;
	
	reg [5:0] index;
	reg [7:0] data;
	reg [1:0] state;
	reg [1:0] delay;
	reg [17:0] count;
	reg RS, halt, sel1;

	wire start;

	

	localparam DELAY0 = 43000/20, DELAY1 = 4300000/20;
	localparam INIT = 0; 
	localparam LINE1 = INIT + 4;
	localparam LINE2 = LINE1 + 11;
	localparam LAST = LINE2 + 14;
	localparam s0 = 0, s1 = 1, s2 = 2, s3=3;

	always @(posedge clk or posedge reset2) begin
		if(reset2) begin
			state <= s0;
			count <= 0;
			index <= INIT;
		end
		else begin
			case (state)
			s0: if(~halt) begin state <= s1; sel1 <= sel; end
			s1: state<=s2;
			s2: if(done) begin
					if (delay) count <= DELAY1;
					else count <= DELAY0;
					state<= s3;
					index <= index + 1;
				end
			s3: if(count == 0) state <= s0;
				else count <= count - 1;
			default: state<=s0;
			endcase
		end
	end
	assign start = (state == s1);

	always @(posedge clk) begin
		data = 8'b0;
		halt = 0;
		delay = 0;
		RS = 1; 
			case (index)
			INIT: begin data=8'b0011_1100; RS=0; delay=1; end
			INIT+1: begin data=8'b0000_1100; RS=0; delay=1; end
			INIT+2: begin data = 8'b00000110; RS=0; delay=1; end
			INIT+3: begin data=8'b0000_0001; RS=0; delay=1; end


			LINE1: begin data = 8'b10000000; RS=0;end
			LINE1+1: data = "2";
			LINE1+2: data = "0";
			LINE1+3: data = "1";
			LINE1+4: data = "8";
			LINE1+5: data = "2";
			LINE1+6: data = "5";
			LINE1+7: data = "3";
			LINE1+8: data = "0";
			LINE1+9: data = "9";
			LINE1+10: data = "6";



			LINE2: begin data = 8'b11000000; RS=0; end
			LINE2+1: data = "R";
			LINE2+2: data = "y";
			LINE2+3: data = "u";
			LINE2+4: data = "S";
			LINE2+5: data = "e";
			LINE2+6: data = "u";
			LINE2+7: data = "n";
			LINE2+8: data = "g";
			LINE2+9: data = "H";
			LINE2+10: data = "y";
			LINE2+11: data = "e";
			LINE2+12: data = "o";
			LINE2+13: data = "n";
			LAST: halt=1;

			default: halt=1;
			endcase
	end
endmodule

	
module lcd_controller(clk, reset2, sel, start, RS, data, done, LCD_RS, LCD_RW, LCD_EN, LCD_DATA);
	input clk, reset2, start, sel;
	input RS;
	input [7:0] data;
	output LCD_RW, LCD_RS, LCD_EN, done;
	output [7:0] LCD_DATA;

	reg [2:0] state;
	reg [3:0] count;
	localparam s0=0, s1=1, s2=2, s3=4, s4=3;
	localparam PW_E = 12;

	assign LCD_RS = RS;
	assign LCD_RW = 1'b0;
	assign LCD_DATA = data;


	always @(posedge clk or posedge reset2) begin
		if (reset2) begin state <= s0; count<=0; end
		else
			case (state)
			s0: if(start) state <= s1;
			s1: state<=s2;
			s2: begin state<=s3; count<=11; end
			s3: if(count==0) state<=s4;
				else count = count - 1;
			s4: state<=s0;
			default: state<=s0;
			endcase
	end
	
	assign LCD_EN = (state==s3);
	assign done = (state==s4);

endmodule


module hex_7seg(dig,seg);	//0이면 켜지고 1이면 꺼짐 
	input [3:0] dig;
	output [6:0] seg;
	reg [6:0] seg;

	always @ (dig)
	case (dig)
			4'h0: seg = 7'b1000000;
			4'h1: seg = 7'b1111001; 	
			4'h2: seg = 7'b0100100; 
			4'h3: seg = 7'b0110000; 	
			4'h4: seg = 7'b0011001; 
			4'h5: seg = 7'b0010010; 	
			4'h6: seg = 7'b0000010; 	
			4'h7: seg = 7'b1111000; 	
			4'h8: seg = 7'b0000000; 	
			4'h9: seg = 7'b0011000; 	
			4'ha: seg = 7'b0001000;
			4'hb: seg = 7'b0000011;
			4'hc: seg = 7'b1000110;
			4'hd: seg = 7'b0100001;
			4'he: seg = 7'b0000110;
			4'hf: seg = 7'b0001110;
	endcase
endmodule

module keyboard(key_clk, key_data, clk50, reset, read, scan_r, scan_c);
	input key_clk;
	input key_data;
	input clk50,reset,read;
	output scan_r;
	output [7:0] scan_c;
	
	reg ready_set;
	reg [7:0] scan_c;
	reg scan_r;
	reg read_char;
	reg clock; 

	reg [3:0] incnt;
	reg [8:0] shiftin;

	reg [7:0] filter;
	reg key_clk_filtered;



	always @ (posedge ready_set or posedge read)
	if (read == 1) scan_r <= 0;
	else scan_r <= 1;


	always @(posedge clk50)
		clock <= ~clock;


	always @(posedge clock)
	begin
		filter <= {key_clk, filter[7:1]};
		if (filter==8'b1111_1111) key_clk_filtered <= 1;
		else if (filter==8'b0000_0000) key_clk_filtered <= 0;
	end


//직렬데이터값읽기

always @(posedge key_clk_filtered)
begin
   if (reset==1)
   begin
      incnt <= 4'b0000;
      read_char <= 0;
   end
   else if (key_data==0 && read_char==0)
   begin
	read_char <= 1;
	ready_set <= 0;
   end
   else
   begin
	   if (read_char == 1)
   		begin
      		if (incnt < 9) 
      		begin
				incnt <= incnt + 1'b1;
				shiftin = { key_data, shiftin[8:1]};
				ready_set <= 0;
			end
		else
			begin
				incnt <= 0;
				scan_c <= shiftin[7:0];
				read_char <= 0;
				ready_set <= 1;
			end
		end
	end
end

endmodule

//펄스값
module p_set(output reg p_out, input trig_in, input clk);
	reg delay;
	always @ (posedge clk)
	begin
		if (trig_in && !delay) p_out <= 1'b1;
		else p_out <= 1'b0;
		delay <= trig_in;
	end 
endmodule


module final(CLOCK_50, KEY, HEX0, HEX1, HEX2, HEX3, HEX4, HEX5, HEX6, HEX7,	PS2_DAT,PS2_CLK, LCD_RS, LCD_RW, LCD_EN, LCD_DATA, LCD_ON, LCD_BLON );

	input CLOCK_50;
	input [3:0] KEY;
	output [6:0] HEX0, HEX1, HEX2, HEX3, HEX4, HEX5, HEX6, HEX7;
	input PS2_DAT;
	input PS2_CLK;

	output LCD_RS, LCD_RW, LCD_EN;
	output [7:0] LCD_DATA;
	output LCD_ON, LCD_BLON;
	

	wire reset = 1'b0;
	wire [7:0] scan_c;

	reg [7:0] h[1:4];
	wire read, scan_r;
	
	wire start, RS, done;
	wire [7:0] data;
	wire clk = CLOCK_50;
	reg sel;

	always @(posedge clk) begin
		sel = KEY[1];
	end

	lcd_test u1(clk, reset, start, RS, data, done, sel);
	lcd_controller(clk, reset, sel, start, RS, data, done, LCD_RS, LCD_RW, LCD_EN, LCD_DATA);

	

	assign LCD_ON = 1'b1;
	assign LCD_BLON = 1'b1;

p_set pulser(
   .p_out(read),
   .trig_in(scan_r),
   .clk(CLOCK_50)
);

keyboard kbd(
  .key_clk(PS2_CLK),
  .key_data(PS2_DAT),
  .clk50(CLOCK_50),
  .reset(reset),
  .read(read),
  .scan_r(scan_r),
  .scan_c(scan_c)
);


//7세그먼트값으로
hex_7seg dsp0(h[1][3:0],HEX0);
hex_7seg dsp1(h[1][7:4],HEX1);

hex_7seg dsp2(h[2][3:0],HEX2);
hex_7seg dsp3(h[2][7:4],HEX3);

hex_7seg dsp4(h[3][3:0],HEX4);
hex_7seg dsp5(h[3][7:4],HEX5);

hex_7seg dsp6(h[4][3:0],HEX6);
hex_7seg dsp7(h[4][7:4],HEX7);



always @(posedge scan_r)
begin
	h[4] <= h[3];
	h[3] <= h[2];
	h[2] <= h[1];
	h[1] <= scan_c;
end

endmodule

