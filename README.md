**Chapter 1: Introduction**

Using Design for Testability (DFT) methodologies, we will improve the IC design in this exercise to make post-fabrication testing easier and more effective. A collection of design approaches and practices called Design for Testability (DFT) aims to make integrated circuits more testable both during production and throughout their operational life. The design will also incorporate Automatic Test Pattern Generation and Scan architecture integration in addition to DFT adjustment. We improve the testability of the IC design by applying these DFT approaches. Post-fabrication testing is made easier, more extensive, and more effective, which enables the early identification of flaws in the design. DFT gives us the potential to enhance the integrated circuit's quality and dependability, providing a more durable and reliable final product. 

**1.1 Goals**

The primary goals of this design are as follows: -

➢ Gated clock and storage elements are the hardest to test, my goal is to modify our design insuch a way I can deal with these elements.

➢ I must integrate a scan architecture into our design.

➢ At the end of this design, I need to generate an automatic test pattern (ATPG) for testing this design.

**Chapter 2: RTL Design**

I must modify my rtl design for testing by adding some scan input and output pins. I also need toconsider the different design styles such as gated clocks.

**2.1 Modifying RTL Design**

In the subfolder rtl, I modified the waveform gen.vhd file. I added the following element to our top RTL design.

➢ SERIAL IN, TESTMODE, and SCAN EN as input ports

➢ SERIAL OUT as output port of the std logic

➢ Memory bypassing, as the state of the memory is unknown, it´s better to bypass.

**2.2 RTL Code**

library IEEE;

use ieee.std_logic_1164.all;

use ieee.numeric_std.all;

use work.parameter.all;

entity waveform_gen is
 port(

 clk,rst,start,mode,data_in : in std_logic;
 
 sync_rst_ram_addr : in std_logic;

period_multiplier:in std_logic_vector(counter_sleep_width-1 downto 0);

 data_out: out std_logic_vector(bit_width_data-1 downto 0);
 
 sleep_reg: out std_logic_vector(counter_sleep_width-1 downto 0);
 
 ---------------------------------------------------------------------------
 
 SERIAL_IN, SCAN_EN, TESTMODE: in std_logic; -- Modified for TST
 
 SERIAL_OUT: out std_logic
 
 );
 
 end entity;
 
architecture behaviour of waveform_gen is

 signal gated_clk,sleep_inv: std_logic;
 
 signal gated_clk2,sleep2_inv: std_logic;
 
 signal re_ram,we_ram,oe_ram,enable_reg_file,sleep,sleep2,enable_shift_reg: std_logic;
 
 signal addr_ram: std_logic_vector(size_ram_addr-1 downto 0);
 
 signal data_shift_reg,ram_out,ram_out_dft:std_logic_vector(bit_width_data-1 downto 0);
 
 -----------------------------------------------------------------------------
 
 signal RAMBO, RAMBO_out : std_logic_vector(bit_width_data-1 downto 0); -- Modified for TST
 
 signal intEnRAM : std_logic;
 
begin

 sleep_inv <= not sleep;
 
 sleep2_inv <= not sleep2;
 
 clk_gate1:CLKGATE_X1
 
 port map(
 
 CK => clk,
 
 E => sleep_inv,
 
 GCK => gated_clk
 );
 
 clk_gate2:CLKGATE_X1
 
 port map(
 
 CK => clk,
 
 E => sleep2_inv,
 
 GCK => gated_clk2
 
 );
 
 controller_main:fsm
 
 port map(
 
 clk=> gated_clk,
 
 rst=>rst,
 
 start=>start,
 
 mode=>mode,
 
 re_ram=>re_ram,
 
 we_ram=>we_ram,
 
 oe_ram=>oe_ram,
 
 enable_shift_reg => enable_shift_reg
 
 );
 
controller_sleep0:controller_sleep

 port map(
 
 clk=>clk,
 
 rst=>rst,
 
 re_ram => re_ram,
 
 we_ram => we_ram,
 
 addr_ram => addr_ram,
 
 sync_rst_ram_counter=> sync_rst_ram_addr,
 
 sleep_time=>period_multiplier,
 
 sleep=>sleep,
 
 sleep2=> sleep2,
 
 enable_reg_file => enable_reg_file,
 
 sleep_reg => sleep_reg
 
 );
 
 shift_reg_data_in:shift_reg
 
 port map(
 
 clk=>gated_clk,
 
 rst=>rst,
 
 data_in=>data_in,
 
 ena=>enable_shift_reg,
 
 data_out=>data_shift_reg
 
 );
 
ram_block:SRAM64x8_1rw

 port map(
 
 CE=>gated_clk,
 
 WEB=>we_ram,
 
 REB=>re_ram,
 
 OEB=>intEnRAM, -- Modified for TST
 
 A=>addr_ram,
 
 I=> data_shift_reg,
 
 O=> ram_out
 
 );
 
 output_register:reg_file
 
 port map(
 
 clk=>gated_clk2,
 
 rst => rst,
 
 ena => enable_reg_file,
 
 data_in => RAMBO_out, -- Modified for TST
 
 data_out => data_out
 
 );
 
 -----------------------------------------------------------------------------
 
 -- Added for TST
 
 -----------------------------------------------------------------------------
 
xor_reg:process(clk,rst) is

 begin
 
 if rst='0' then
 
 RAMBO <= (others => '0');
 
 elsif clk='1' and clk'event then
 
 RAMBO <= ((addr_ram & we_ram & re_ram) xor data_shift_reg);
 
 end if;
 
 end process xor_reg;
 
mux2:process(RAMBO ,ram_out, TESTMODE) is

 begin
 
 if TESTMODE='1' then
 
 RAMBO_out <= RAMBO;
 
 else
 
 RAMBO_out <= ram_out;
 
 end if;
 
 end process mux2;
 
mux1:process(oe_ram, TESTMODE) is

 begin
 
 if TESTMODE='1' then
 
 intEnRAM <= '0';
 
 else
 
 intEnRAM <= oe_ram;
 
 end if;
 
 end process mux1;
 
end architecture; 


**Chapter 3: Launching Design Compiler**

To use the Design compiler, we created a sourceme.sh file with the following content in the do_synth run folder.

export SNPSLMD_LICENSE_FILE=28231@item0096 

export PATH=/usrf01/prog/synopsys/syn/R-2020.09-SP4/bin:${PATH} 

Now I can source the file by writing source.sourceme.sh 

To use the standard cell libraries and to instruct the tool to save the log files into the directories that I have previously defined we created a file named .synopsys_dc.setup with the following contents. 

define_design_lib work -path ./tool/work 

set_app_var  view_log_file    

./log/synth_view.log 

set_app_var  sh_command_log_file  ./log/synth_sh.log 

set_app_var  filename_log_file    ./log/synth_file.log 

set_app_var search_path    

[concat ./cmd/  [get_app_var search_path] ] 

set library_path "../../../0_FreePDK45/LIB/" 

set library_name "NangateOpenCellLibrary_typical_ccs_scan.db" 

set library_name2 "SRAM64x8_1rw.db" 

set_app_var target_library    $library_name 

set_app_var link_library    

[concat $library_name $library_name2 dw_foundation.sldb "*"] 

set_app_var search_path       

[concat $library_path [get_app_var search_path] ] 

set_app_var synthetic_library [list dw_foundation.sldb] 

set_app_var symbol_library    [list class.sdb] 

set_app_var vhdlout_use_packages { ieee.std_logic_1164.all  NangateOpenCellLibrary.Components.all } 

set_app_var vhdlout_write_components FALSE 

Afterward, I launch the design compiler with the command design_vision | tee log/synthesis.log, tee log/synthesis.log command is used to make sure that a log file is saved in the file synthesis.log in the log directory. 

**Chapter 4: DFT configuration and synthesis**

**4.1 Importing Design**

Before DFT synthesis, I write some TCL commands to set different options in the tool. For a more complex design, it is more useful to write the TCL commands in a separate file and then later source 
that file for use. But for our purpose, we can source them separately. The TCL codes have the 
following functions: - 

➢ config_synth.tcl: all .vhd files, top-level rtl files, names, and variables are defined in this file. 

➢ read_design.tcl: used to read the RTL code. 

➢ global_constraints.tcl: used to set simple design constraints.  

We can source the TCL command by writing source ./cmd/config_synth.tcl, source ./cmd/read_design.tcl, source ./cmd/global_constraints.tcl.  

**4.2 First DFT Synthesis**

To synthesis the DFT, first, I am going to configure the DFT. Several settings must be set, to configure the DFT. For example, I choose the clock, test duration, kind of scan cells, reset, scan 
input and output, scan enable, and test mode from all the possibilities. For scan cells, there are three widely used scan cells Muxed-D, Clocked, and LSSD scan cells. For our design, we used Muxed-D scan cells.  

I created a file called dft_config.tcl to add all the dftsynthesis commands by writing emacs 

cmd/dft_config.tcl. 

Into that file, we added the following DFT configuration commands: 

set_scan_configuration -style multiplexed_flip_flop 

set test_default_period 100 

set_dft_signal -view existing_dft -type ScanClock -timing {45 55} -port clk 

set_dft_signal -view existing_dft -type Reset -active_state 0 -port rst 

set_dft_signal -view spec -type ScanDataIn -port SERIAL_IN 

set_dft_signal -view spec -type ScanDataOut -port SERIAL_OUT 

set_dft_signal -view spec -type ScanEnable -port SCAN_EN -active_state 1 

set_dft_signal -view existing_dft -type Constant -port TESTMODE -active_state 1 

create_test_protocol 

After that we started DFT synthesis by executing the command: compile -scan, with this we compiled 

our system using the scan option. This tool replaces all the sequential elements with scan equivalent. 

Now, we can check the design for DFT violations using the command: dft_drc.  

After that to specify the scan chain we wrote the following commands: 

➢ Set_scan_configuration -chain_count 1, to select the number of the scan chain. 

➢ Set_scan_configuration -clock_mixing no_mix, to define that no different clock edges may occur in the scan chain. 

➢ set_scan_path chain1 -scan_data_in SERIAL_IN -scan_data_out SERIAL_OUT, to define the input and output of the scan chain. 

➢ Insert_dft, to insert the scan chain into the design. 

➢ Set_scan_state scan_existing, indicates if the scan chain is fully implemented or not. 

Now to see the report of the scan chain 

report_scan_path -view existing_dft -chain all > reports/chain1.rep 

report_scan_path -view existing_dft -cell all > reports/cell1.rep 

**4.3 Second DFT synthesis**

From the RTL of our design, I can see that, I have a gated clock in my design, to improve the testability of this design I need to do further modifications. So, I replaced our gated clock architecture with its test equivalent architecture.  

The modified RTL code looks like:  

clk_gate1:CLKGATETST_X1    

port map( 

CK => clk, 

E => sleep_inv, 

GCK => gated_clk,

SE => TESTMODE 

); 

clk_gate2:CLKGATETST_X1     

port map(

CK => clk,

E => sleep2_inv,

GCK => gated_clk2,

SE => TESTMODE

); -- TEST CLOCK GATE -- TEST CLOCK GATE

With this standard cell, I use an extra OR gate, which forces CEN to 1 using either the TM or SE signals, which solves my gated clock problem for testability.  

Now I need to do the DFT configurations again for new RTL code. I can source my 

dft_config.tcl file by writing source ./cmd/dft_config.tcl 

After the configuration, I can start my DFT synthesis by running compile -scan, and check the DFT 

violation by writing dft_drc, same as before. 

I received no design violations, as a result, which represents our design is good.  

Now I can specify the scan chain, same as before by writing the following codes. 

Set_scan_configuration -chain_count 1 

Set_scan_configuration -clock_mixing no_mix 

Set_scan_path chain1 -scan_data_in SERIAL_IN -scan_data_out SERIAL_OUT 

Insert_dft 

Set_scan_state scan_existing 

Now to see the report of the scan chain 

Report_scan_path -view existing_dft -chain all > reports/chain1.rep 

Report_scan_path -view existing_dft -cell all > reports/cell1.rep 

Now I save my files for test pattern generation with TetraMax, using the following tcl commands. 

Change_names -hierarchy -rule verilog 

Write -format verilog -hierarchy -out results/waveform_gen.vg 

Write -format ddc -hierarchy -output results/waveform_gen.ddc 

Write_scan_def -output results/waveform_gen.def 

Set test_stil_netlist_format verilog 

Write_test_protocol -output results/waveform_gen.stil 

After that, I used remove_design command the remove the design. 

**Chapter 5: Automatic Test Pattern Generation**

**5.1 Lunching TetraMax**

To create a test pattern, I need to use TetraMax, after going to the TetraMax subfolder, I can launch TetraMax by using The following command. 

Export PATH=”/usrf01/prog/synopsys/txs/R-2020.09-SP4/bin:${PATH}”

Tmax

![Screenshot 2024-12-21 124813](https://github.com/user-attachments/assets/1987aa30-9a8e-4970-b7e6-7d1e414f7ca3)

**5.2 Reading Netlist and Library Files**

Now I have to read the netlist and the library file into the read Netlist window. I found the netlist in do_synth/results sub-directory. I also had to add the nangate library file. 


![Screenshot 2024-12-21 125054](https://github.com/user-attachments/assets/65a4b6e2-8d92-44cd-8dc1-4ed6131d1096)

**5.3 Building the Design**

To build this design I need to set relevant parameters. First, I select build from the menu bar and select this top model. I used the default options of the Set learning box.

![Screenshot 2024-12-21 125245](https://github.com/user-attachments/assets/c1f22914-a990-46a1-8bce-1054cd0011bb)


After selecting the top module, I selected Set Build Options and added SRAM as the black box. 

![Screenshot 2024-12-21 125353](https://github.com/user-attachments/assets/09f5e538-2585-4527-83bc-1b6becbc17c8)

5.4. DRC check 
Now, I checked the design rules (DRC), and for that, I went to the DRC window and selected the test protocol file named waveform_gen.stil. This is the file I created and saved in the previous step in the do_synth/result sub-folder. Now I can initiate the DRC check by clicking run.  

![Screenshot 2024-12-21 125707](https://github.com/user-attachments/assets/6c669426-c2dc-4947-a223-dda786a04830)

The DRC check result looks like this, which implies there were no design violations. 

![Screenshot 2024-12-21 125907](https://github.com/user-attachments/assets/b98afbfb-15c0-49bb-a16e-b0b288ce413f)

**5.5. Generate the Test Pattern**

Now after importing the design and completing the DRC check without error, I used TetraMax to create the test patterns. We used the following two commands to annotate the errors and generate the 
test patterns: 

add_faults -all 

run_atpg -auto 

![Screenshot 2024-12-21 130024](https://github.com/user-attachments/assets/5f6a27df-5222-4c55-a3b8-3a01002f9797)

As I can see from my result, I got 92.04% test coverage. Finally, I saved my generated test pattern in binary format with following the command: 

write_patterns waveform_gen_pattern.v -internal -format binary




