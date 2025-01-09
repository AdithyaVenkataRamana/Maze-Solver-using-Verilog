# Maze-Solver-using-Verilog
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
