# FPGA-based-acceleration-for-CNN
#libe buffer
`timescale 1ns / 1ps

module line_buffer #(
    parameter IMG_WIDTH = 32, 
    parameter DATA_WIDTH = 8
)(
    input wire clk,
    input wire rst_n,
    input wire valid_in,
    input wire [DATA_WIDTH-1:0] pixel_in, // Unsigned
    
    output reg [DATA_WIDTH-1:0] w00, w01, w02,
    output reg [DATA_WIDTH-1:0] w10, w11, w12,
    output reg [DATA_WIDTH-1:0] w20, w21, w22,
    output reg data_valid_out
);

    reg [DATA_WIDTH-1:0] line_fifo_0 [0:IMG_WIDTH-1];
    reg [DATA_WIDTH-1:0] line_fifo_1 [0:IMG_WIDTH-1];
    integer i;
    reg [15:0] pixel_counter;
    localparam WAIT_CYCLES = (2 * IMG_WIDTH) + 2;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            for (i=0; i<IMG_WIDTH; i=i+1) begin
                line_fifo_0[i] <= 0; line_fifo_1[i] <= 0;
            end
            w00<=0; w01<=0; w02<=0; w10<=0; w11<=0; w12<=0; w20<=0; w21<=0; w22<=0;
            pixel_counter <= 0; data_valid_out <= 0;
        end else if (valid_in) begin
            for (i=IMG_WIDTH-1; i>0; i=i-1) begin
                line_fifo_1[i] <= line_fifo_1[i-1];
                line_fifo_0[i] <= line_fifo_0[i-1];
            end
            line_fifo_1[0] <= line_fifo_0[IMG_WIDTH-1];
            line_fifo_0[0] <= pixel_in;

            w02 <= pixel_in;              w01 <= w02; w00 <= w01;
            w12 <= line_fifo_0[IMG_WIDTH-1]; w11 <= w12; w10 <= w11;
            w22 <= line_fifo_1[IMG_WIDTH-1]; w21 <= w22; w20 <= w21;

            if (pixel_counter < WAIT_CYCLES) begin
                pixel_counter <= pixel_counter + 1;
                data_valid_out <= 0;
            end else data_valid_out <= 1;
        end else data_valid_out <= 0;
    end
endmodule
