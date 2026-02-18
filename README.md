# FPGA-based-acceleration-for-CNN

`timescale 1ns / 1ps

module convolution2d #(
    parameter IMG_WIDTH = 32
)(
    input wire clk,
    input wire rst_n,
    input wire valid_in,
    input wire [7:0] pixel_in, // REMOVED "signed" here
    input wire signed [7:0] weight_00, weight_01, weight_02,
    input wire signed [7:0] weight_10, weight_11, weight_12,
    input wire signed [7:0] weight_20, weight_21, weight_22,
    output wire signed [15:0] conv_result,
    output reg valid_out
);

    wire [7:0] w00, w01, w02, w10, w11, w12, w20, w21, w22;
    wire lb_valid;

    line_buffer #(.IMG_WIDTH(IMG_WIDTH)) lb_inst (
        .clk(clk), .rst_n(rst_n), .valid_in(valid_in), .pixel_in(pixel_in),
        .w00(w00), .w01(w01), .w02(w02),
        .w10(w10), .w11(w11), .w12(w12),
        .w20(w20), .w21(w21), .w22(w22),
        .data_valid_out(lb_valid)
    );

    muladdtree3x3 mat_inst (
        .clk(clk), .rst_n(rst_n),
        .p00(w20), .p01(w21), .p02(w22),
        .p10(w10), .p11(w11), .p12(w12),
        .p20(w00), .p21(w01), .p22(w02),
        .w00(weight_00), .w01(weight_01), .w02(weight_02),
        .w10(weight_10), .w11(weight_11), .w12(weight_12),
        .w20(weight_20), .w21(weight_21), .w22(weight_22),
        .result_out(conv_result)
    );

    reg [3:0] valid_pipe;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            valid_pipe <= 0; valid_out <= 0;
        end else begin
            valid_pipe <= {valid_pipe[2:0], lb_valid};
            valid_out <= valid_pipe[3];
        end
    end
endmodule
