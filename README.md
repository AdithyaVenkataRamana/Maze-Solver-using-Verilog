# Maze-Solver-using-Verilog
```
	module mazesolver_1 #(
    parameter maze_col = 4,  // Changed to 4 for debugging
    parameter maze_row = 4,  // Changed to 4 for debugging
    parameter start_point_x = 0,  // X coordinate of the start point
    parameter start_point_y = 0,  // Y coordinate of the start point
    parameter end_point_x = 3,  // X coordinate of the end point
    parameter end_point_y = 3   // Y coordinate of the end point
)(
    input wire clk,
    input wire rst,
    input wire start,
    input wire [maze_col*maze_row-1:0] maze,
    output wire [maze_col*maze_row-1:0] solution,
    output wire done
);
  // Function to calculate the ceiling of log2 of a number
function integer clogb2(input integer value);
    integer result;
    begin
        result = 0;
        while (2**result < value) begin
            result = result + 1;
        end
        clogb2 = result;
    end
endfunction


// Matrix for Maze
wire maze_matrix [maze_row-1:0][maze_col-1:0];
reg visited_matrix [maze_row-1:0][maze_col-1:0];
reg solution_matrix [maze_row-1:0][maze_col-1:0];

// Define position coordinate
reg [clogb2(maze_col)-1:0] curr_x, prev_x;
reg [clogb2(maze_row)-1:0] curr_y, prev_y;

reg track_back_upd;

// Start rising edge detect
reg start_dly;
wire start_rising_edge;

reg [1:0] done_temp;

integer i,j;

genvar k,x;
generate 
    for (k = 0; k < maze_row; k = k + 1) begin
       for (x = 0; x < maze_col; x = x + 1) begin
          assign maze_matrix[k][x] = maze[k*maze_col + x];
       end
    end
endgenerate

always @(posedge clk or posedge rst) begin
    if (rst) begin
        start_dly <= 1'b0;
    end
    else begin
        start_dly <= start;
    end
end

assign start_rising_edge = ~start_dly & start;

always @(posedge clk or posedge rst) begin
    if (rst) begin
        // Reset the maze solver state
        curr_x <= 0;
        curr_y <= 0;
        prev_x <= 0;
        prev_y <= 0;
        track_back_upd <= 0;
    end else if(start_rising_edge) begin 
        curr_x <= start_point_x;
        curr_y <= start_point_y;
        $display("Starting solver at (%0d, %0d)", curr_x, curr_y);  // Debug output
    end else if(!start) begin
        curr_x <= 0;
        curr_y <= 0;
        prev_x <= 0;
        prev_y <= 0;
        track_back_upd <= 0;
    end else if(curr_x != end_point_x | curr_y != end_point_y) begin
        prev_x <= curr_x;
        prev_y <= curr_y;
        // Move to the next unvisited and unobstructed neighbor (right, down, left, up)
        // Move right
        if (curr_x < (maze_col - 1) & !visited_matrix[curr_x + 1][curr_y] & maze_matrix[curr_x + 1][curr_y]) begin
            curr_x <= curr_x + 1;
            track_back_upd <= 0;
        // Move down
        end else if (curr_y < (maze_row - 1) & !visited_matrix[curr_x][curr_y + 1] & maze_matrix[curr_x][curr_y + 1]) begin
            curr_y <= curr_y + 1;
            track_back_upd <= 0;
        // Move left
        end else if (curr_x > 0 & !visited_matrix[curr_x - 1][curr_y] & maze_matrix[curr_x - 1][curr_y]) begin
            curr_x <= curr_x - 1;
            track_back_upd <= 0;
        // Move up
        end else if (curr_y > 0 & !visited_matrix[curr_x][curr_y - 1] & maze_matrix[curr_x][curr_y - 1]) begin
            curr_y <= curr_y - 1;
            track_back_upd <= 0;
        // Track back
        end else begin
            track_back_upd <= 1;
            
            // track back right
            if (curr_x < (maze_col - 1) & solution_matrix[curr_x + 1][curr_y])
                curr_x <= curr_x + 1;
            // track back down
            else if (curr_y < (maze_row - 1) & solution_matrix[curr_x][curr_y + 1]) 
                curr_y <= curr_y + 1;
            // track back left
            else if (curr_x > 0 & solution_matrix[curr_x - 1][curr_y]) 
                curr_x <= curr_x - 1;
            // track back up
            else if (curr_y > 0 & solution_matrix[curr_x][curr_y - 1]) 
                curr_y <= curr_y - 1;
            
        end        
    end
end

// solution_matrix
always @(posedge clk or posedge rst) begin
    if (rst) begin
        for (i = 0; i < maze_row; i = i + 1) begin
           for (j = 0; j < maze_col; j = j + 1) begin
              solution_matrix[i][j] <= 1'b0;
           end
        end
    end else if (!start) begin
        for (i = 0; i < maze_row; i = i + 1) begin
           for (j = 0; j < maze_col; j = j + 1) begin
              solution_matrix[i][j] <= 1'b0;
           end
        end
    end else if (track_back_upd) begin
        solution_matrix[curr_x][curr_y] <= 1'b1;
    end
end

assign done = (curr_x == end_point_x && curr_y == end_point_y);

endmodule

	```
   ```




```
	module mazesolver_tb;
    // Parameters for the maze
    parameter MAZE_ROWS = 4;   // Changed to 4x4 for quicker debugging
    parameter MAZE_COLS = 4;   
    parameter STREAM_WIDTH = MAZE_ROWS * MAZE_COLS;
    parameter MAX_CYCLES = 5000;  // Maximum allowed cycles for timeout

    // Testbench signals
    reg clk;
    reg rst;
    reg start;
    reg [STREAM_WIDTH-1:0] maze;  // Flattened maze input
    wire done;
    wire [STREAM_WIDTH-1:0] solution;  // Flattened solution output

    // Cycle counter to monitor timeouts
    integer cycle_count;

    // DUT instantiation
    mazesolver_1 dut (
        .clk(clk),
        .rst(rst),
        .start(start),
        .maze(maze),
        .done(done),
        .solution(solution)
    );

    // Clock generation
    initial begin
        clk = 0;
        forever #50 clk = ~clk;  // 100ns clock period
    end

    // Task to initialize the maze
    task initialize_maze(input integer rows, input integer cols, 
                         output reg [STREAM_WIDTH-1:0] flat_maze);
        integer i, j;
        begin
            flat_maze = 0;
            for (i = 0; i < rows; i = i + 1) begin
                for (j = 0; j < cols; j = j + 1) begin
                    if ((i == 0 && j == 0) || (i == rows-1 && j == cols-1)) begin
                        flat_maze[i * cols + j] = 1;  // Start and end points
                    end else begin
                        flat_maze[i * cols + j] = $random % 2;  // Randomize maze cells
                    end
                end
            end
        end
    endtask

    // Testbench
    initial begin
        reg [STREAM_WIDTH-1:0] flat_maze;
        reg [STREAM_WIDTH-1:0] flat_solution;

        // Reset
        rst = 1;
        start = 0;
        maze = 0;
        cycle_count = 0;  // Reset cycle counter
        #100;  // Wait for reset duration
        rst = 0;

        // Generate random maze and send to DUT
        initialize_maze(MAZE_ROWS, MAZE_COLS, flat_maze);
        maze = flat_maze;

        // Start signal
        start = 1;
        #100;  // Wait for start signal duration
        start = 0;

        // Wait for DUT to complete or timeout
        while (!done && cycle_count < MAX_CYCLES) begin
            cycle_count = cycle_count + 1;
            #100;  // Wait for next clock cycle
        end

        if (cycle_count >= MAX_CYCLES) begin
            $display("Simulation Timeout: Maze solver did not complete in time.");
            $stop;
        end

        // Capture solution
        flat_solution = solution;

        // Display maze and solution
        $display("Maze:");
        for (integer i = 0; i < MAZE_ROWS; i = i + 1) begin
            $write("Row %0d: ", i);
            for (integer j = 0; j < MAZE_COLS; j = j + 1) begin
                $write("%b ", flat_maze[i * MAZE_COLS + j]);
            end
            $write("\n");
        end

        $display("Solution:");
        for (integer i = 0; i < MAZE_ROWS; i = i + 1) begin
            $write("Row %0d: ", i);
            for (integer j = 0; j < MAZE_COLS; j = j + 1) begin
                $write("%b ", flat_solution[i * MAZE_COLS + j]);
            end
            $write("\n");
        end

        // End simulation
        $stop;
    end
  initial begin
    $dumpfile("dump.vcd"); $dumpvars;
  end
endmodule

	```
   ```
