CROSS     = riscv64-unknown-elf-
SOFTWARE  = .template/cpu/*.cpp .packages/*/*.cpp micon/*.cpp software/*.cpp
HARDWARE  = .template/fpga/*.v .template/cpu/*.v .packages/*/*.v micon/*.v
SIMU_HARD = .template/simulation/*.v
SIMU_SOFT = .template/cpu/*.cpp .packages/*/*.cpp micon/*.cpp simulation/*.cpp
PCF       = .template/fpga/*.pcf
START     = .template/cpu/start.S
LINKER    = .template/cpu/sections.lds

upload: hardware software
	tinyprog -p .build/hardware.bin -u .build/software.bin

install:
	micon install -m micon.mcl

gen: micon/hardware.v micon/firmware.hpp micon/firmware.cpp
	micon read -m micon.mcl
micon/hardware.v: micon.mcl
	micon gen-hard -t $@ -m $^ -o $@
micon/firmware.hpp: micon.mcl
	micon gen-firm -t $@ -m $^ -o $@
micon/firmware.cpp: micon.mcl
	micon gen-firm -t $@ -m $^ -o $@

hardware: .build/hardware.bin
.build/hardware.json: $(HARDWARE)
	yosys -ql $@.log -p 'synth_ice40 -top hardware -json $@' $^
.build/hardware.asc: .build/hardware.json
	nextpnr-ice40 -ql $@.log --lp8k --package cm81 --asc $@ --pcf $(PCF) --json $^
.build/hardware.bin: .build/hardware.asc
	icetime -d lp8k -c 12 -mtr .build/hardware.rpt $^
	icepack $^ $@

software: .build/software.objdump .build/software.nm .build/software.bin
.build/software.elf: $(START) $(SOFTWARE)
	$(CROSS)g++ -march=rv32imc -mabi=ilp32 -nostartfiles \
	        -Wl,-Bstatic,-T,$(LINKER),--strip-debug,-Map=.build/software.map,--cref \
			-O3 -ffreestanding -nostdlib -I .template/cpu -I .packages -I micon -o $@ $^
.build/software.objdump: .build/software.elf
	$(CROSS)objdump --demangle -D $^ > $@
.build/software.nm: .build/software.elf
	$(CROSS)nm --demangle --numeric-sort $^ > $@
.build/software.bin: .build/software.elf
	$(CROSS)objcopy -O binary $^ /dev/stdout > $@

.build/simu_software.elf: $(START) $(SIMU_SOFT)
	$(CROSS)g++ -march=rv32imc -mabi=ilp32 -nostartfiles \
	        -Wl,-Bstatic,-T,$(LINKER),--strip-debug,-Map=.build/simu_software.map,--cref \
			-O3 -ffreestanding -nostdlib -I .template/cpu -I .packages -I micon -o $@ $^ \
			-DSIMU
.build/simu_software.objdump: .build/simu_software.elf
	$(CROSS)objdump --demangle -D $^ > $@
.build/simu_software.nm: .build/simu_software.elf
	$(CROSS)nm --demangle --numeric-sort $^ > $@
.build/simu_software.bin: .build/simu_software.elf
	$(CROSS)objcopy -O binary $^ $@
.build/simu_software.hex: .build/simu_software.bin
	xxd $^ > $@
.build/simu_flash.bin: .build/simu_software.bin
	sh .template/simulation/zeropadding.sh $^ > $@
.build/simu_flash.hex: .build/simu_flash.bin
	xxd -c 1 -p $^ > $@
.build/simu_testbench.vvp: .build/simu_flash.hex $(SIMU_HARD) $(HARDWARE)
	iverilog -g2005-sv -s testbench -o $@ $(SIMU_HARD) $(HARDWARE) \
	         `yosys-config --datdir/ice40/cells_sim.v` \
			 -DNO_ICE40_DEFAULT_ASSIGNMENTS \
			 -DDEBUG -DDEBUGNETS -DDEBUGREGS
.build/simulation.vcd: .build/simu_testbench.vvp
	vvp $^ > .build/simulation.log
	sh .template/simulation/serial.sh .build/simulation.log .build/simu_serial.log
simu: .build/simulation.vcd .build/simu_software.objdump .build/simu_software.nm
	gtkwave .build/simulation.vcd

.PHONY: upload install gen hardware software simu
