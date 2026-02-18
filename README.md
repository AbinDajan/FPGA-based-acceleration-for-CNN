# FPGA-based-acceleration-for-CNN

`timescale 1ns / 1ps

module tb_conv2d();

    reg clk;
    reg rst_n;
    reg valid_in;
    reg [7:0] pixel_in; // Unsigned [7:0]
    
    reg [7:0] w_mem [0:8];
    reg [7:0] img_mem [0:1023]; 

    reg signed [7:0] w[0:2][0:2];
    wire signed [15:0] conv_result;
    wire valid_out;
    reg signed [15:0] max_val_hw;

    convolution2d #(.IMG_WIDTH(32)) dut (
        .clk(clk), .rst_n(rst_n), .valid_in(valid_in), .pixel_in(pixel_in),
        .weight_00(w[0][0]), .weight_01(w[0][1]), .weight_02(w[0][2]),
        .weight_10(w[1][0]), .weight_11(w[1][1]), .weight_12(w[1][2]),
        .weight_20(w[2][0]), .weight_21(w[2][1]), .weight_22(w[2][2]),
        .conv_result(conv_result), .valid_out(valid_out)
    );

    always #5 clk = ~clk;

    // Max Value Logic
    always @(posedge clk) begin
        if (!rst_n) max_val_hw <= -32768; 
        else if (valid_out && conv_result > max_val_hw) max_val_hw <= conv_result;
    end

    integer i;
    initial begin
        clk = 0; rst_n = 0; valid_in = 0; pixel_in = 0; max_val_hw = -32768;

        // -------------------------------------------------------------
        // PASTE YOUR FULL PATHS BELOW (Use / not \)
        // -------------------------------------------------------------
        $readmemh("C:/Users/abind/OneDrive/Desktop/project/cnn vivado/weights.txt", w_mem);
        $readmemh("C:/Users/abind/OneDrive/Desktop/project/cnn vivado/image.txt", img_mem);
        // -------------------------------------------------------------

        // Assign Weights
        w[0][0]=$signed(w_mem[0]); w[0][1]=$signed(w_mem[1]); w[0][2]=$signed(w_mem[2]);
        w[1][0]=$signed(w_mem[3]); w[1][1]=$signed(w_mem[4]); w[1][2]=$signed(w_mem[5]);
        w[2][0]=$signed(w_mem[6]); w[2][1]=$signed(w_mem[7]); w[2][2]=$signed(w_mem[8]);

        #100; rst_n = 1; #20;

        for (i = 0; i < 1024; i = i + 1) begin
            @(posedge clk); valid_in = 1; pixel_in = img_mem[i]; 
        end
        @(posedge clk); valid_in = 0; pixel_in = 0;
        
        #2000;
        $display("\nMAX VALUE: %d\n", max_val_hw);
        $stop;
    end
endmodule

