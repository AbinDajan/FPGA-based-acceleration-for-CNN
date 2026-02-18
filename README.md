# FPGA-based-acceleration-for-CNN

`timescale 1ns / 1ps

module muladdtree3x3 (
    input wire clk,
    input wire rst_n,
    // PIXELS: Unsigned (0 to 255)
    input wire [7:0] p00, p01, p02,
    input wire [7:0] p10, p11, p12,
    input wire [7:0] p20, p21, p22,
    // WEIGHTS: Signed (-128 to 127)
    input wire signed [7:0] w00, w01, w02,
    input wire signed [7:0] w10, w11, w12,
    input wire signed [7:0] w20, w21, w22,
    output reg signed [15:0] result_out
);

    reg signed [15:0] mult_00, mult_01, mult_02;
    reg signed [15:0] mult_10, mult_11, mult_12;
    reg signed [15:0] mult_20, mult_21, mult_22;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            mult_00 <= 0; mult_01 <= 0; mult_02 <= 0;
            mult_10 <= 0; mult_11 <= 0; mult_12 <= 0;
            mult_20 <= 0; mult_21 <= 0; mult_22 <= 0;
        end else begin
            // CONVERT UNSIGNED PIXEL TO SIGNED before multiply
            // We append a 0 bit at the front: {1'b0, p00}
            mult_00 <= $signed({1'b0, p00}) * w00;
            mult_01 <= $signed({1'b0, p01}) * w01;
            mult_02 <= $signed({1'b0, p02}) * w02;
            mult_10 <= $signed({1'b0, p10}) * w10;
            mult_11 <= $signed({1'b0, p11}) * w11;
            mult_12 <= $signed({1'b0, p12}) * w12;
            mult_20 <= $signed({1'b0, p20}) * w20;
            mult_21 <= $signed({1'b0, p21}) * w21;
            mult_22 <= $signed({1'b0, p22}) * w22;
        end
    end

    // Adder Trees
    reg signed [16:0] add_st1_0, add_st1_1, add_st1_2, add_st1_3, add_st1_rem;
    reg signed [17:0] add_st2_0, add_st2_1, add_st2_rem;
    reg signed [19:0] final_sum;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            add_st1_0 <= 0; add_st1_1 <= 0; add_st1_2 <= 0; add_st1_3 <= 0; add_st1_rem <= 0;
        end else begin
            add_st1_0 <= mult_00 + mult_01;
            add_st1_1 <= mult_02 + mult_10;
            add_st1_2 <= mult_11 + mult_12;
            add_st1_3 <= mult_20 + mult_21;
            add_st1_rem <= mult_22;
        end
    end

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            add_st2_0 <= 0; add_st2_1 <= 0; add_st2_rem <= 0;
        end else begin
            add_st2_0 <= add_st1_0 + add_st1_1;
            add_st2_1 <= add_st1_2 + add_st1_3;
            add_st2_rem <= add_st1_rem;
        end
    end

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) result_out <= 0;
        else begin
            final_sum = add_st2_0 + add_st2_1 + add_st2_rem;
            result_out <= final_sum[15:0];
        end
    end
endmodule
