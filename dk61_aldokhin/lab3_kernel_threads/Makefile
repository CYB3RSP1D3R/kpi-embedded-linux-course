# (!) using paths with spaces may not work with Kbuild
INCDIR := inc
SRCDIR := src
obj-m := lab3.o

# this is what is used by Kbuild
obj-m += $(addprefix $(SRCDIR)/,$(OBJ))

EXTRA_CFLAGS += -I$(INCDIR) -std=gnu11

# directory containing Makefile for kernel build
KBUILDDIR := /lib/modules/$(shell uname -r)/build/

.PHONY: modules clean tidy all run

all: | tidy modules run

.build:
	cp -r ./$(SRCDIR) ./$@
	cp Makefile $@/

# recur to the original kernel Makefile with some additions
modules: .build
	cd $< && $(MAKE) -C "$(KBUILDDIR)" M="$(PWD)/$<" modules

tidy:
	-rm -rf .build

# we link original clean to target named tidy
.ONESHELL:
clean:
	cd .build
	-rm -f modules.order Module.symvers
	-rm -rf .tmp_versions
	-rm -f $(obj-m) $(obj-m:.o=.mod.o) $(obj-m:.o=.mod.c)
	-rm -f $(addsuffix .cmd,$(addprefix .,$(obj-m)))
	-rm -f $(addsuffix .cmd,$(addprefix .,$(obj-m:.o=.ko)))
	-rm -f $(addsuffix .cmd,$(addprefix .,$(obj-m:.o=.mod.o)))

.ONESHELL:
#run:
#	sudo sh -c "insmod .build/$(obj-m:.o=.ko) iterator=10 kernel_quantity=10; "
#sleep 1; rmmod $(obj-m:.o=); sleep 1; dmesg -k | tail -10"

	
