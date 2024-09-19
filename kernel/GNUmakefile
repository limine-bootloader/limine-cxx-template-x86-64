# Nuke built-in rules and variables.
MAKEFLAGS += -rR
.SUFFIXES:

# This is the name that our final executable will have.
# Change as needed.
override OUTPUT := kernel

# Convenience macro to reliably declare user overridable variables.
override USER_VARIABLE = $(if $(filter $(origin $(1)),default undefined),$(eval override $(1) := $(2)))

# Destination directory on install (should always be empty by default).
$(call USER_VARIABLE,DESTDIR,)

# Install prefix; /usr/local is a good, standard default pick.
$(call USER_VARIABLE,PREFIX,/usr/local)

# User controllable C compiler command.
$(call USER_VARIABLE,KCC,cc)

# User controllable C++ compiler command.
$(call USER_VARIABLE,KCXX,c++)

# User controllable linker command.
$(call USER_VARIABLE,KLD,ld)

# User controllable C flags.
$(call USER_VARIABLE,KCFLAGS,-g -O2 -pipe)

# User controllable C++ flags. We default to same as C flags.
$(call USER_VARIABLE,KCXXFLAGS,$(KCFLAGS))

# User controllable C/C++ preprocessor flags. We set none by default.
$(call USER_VARIABLE,KCPPFLAGS,)

# User controllable nasm flags.
$(call USER_VARIABLE,KNASMFLAGS,-F dwarf -g)

# User controllable linker flags. We set none by default.
$(call USER_VARIABLE,KLDFLAGS,)

# Ensure the dependencies have been obtained.
ifneq ($(shell ( test '$(MAKECMDGOALS)' = clean || test '$(MAKECMDGOALS)' = distclean ); echo $$?),0)
    ifeq ($(shell ( ! test -d freestnd-c-hdrs || ! test -d freestnd-cxx-hdrs || ! test -f src/cc-runtime.c || ! test -f src/limine.h ); echo $$?),0)
        $(error Please run the ./get-deps script first)
    endif
endif

# Check if KCC is Clang.
override KCC_IS_CLANG := $(shell ! $(KCC) --version | grep 'clang' >/dev/null 2>&1; echo $$?)

# Check if KCXX is Clang.
override KCXX_IS_CLANG := $(shell ! $(KCXX) --version | grep 'clang' >/dev/null 2>&1; echo $$?)

# If the C compiler is Clang, set the target as needed.
ifeq ($(KCC_IS_CLANG),1)
    override KCC += \
        -target x86_64-unknown-none
endif

# Likewise for the C++ compiler.
ifeq ($(KCXX_IS_CLANG),1)
    override KCXX += \
        -target x86_64-unknown-none
endif

# Internal flags shared by both C and C++ compilers.
override SHARED_FLAGS := \
    -Wall \
    -Wextra \
    -nostdinc \
    -ffreestanding \
    -fno-stack-protector \
    -fno-stack-check \
    -fno-lto \
    -fno-PIC \
    -ffunction-sections \
    -fdata-sections \
    -m64 \
    -march=x86-64 \
    -mno-80387 \
    -mno-mmx \
    -mno-sse \
    -mno-sse2 \
    -mno-red-zone \
    -mcmodel=kernel

# Internal C/C++ preprocessor flags that should not be changed by the user.
override KCPPFLAGS_C := \
    -I src \
    -isystem freestnd-c-hdrs/x86_64/include \
    $(KCPPFLAGS) \
    -MMD \
    -MP

# Internal C++ only preprocessor flags.
override KCPPFLAGS_CXX := \
    -I src \
    -isystem freestnd-c-hdrs/x86_64/include \
    -isystem freestnd-cxx-hdrs/x86_64/include \
    $(KCPPFLAGS) \
    -MMD \
    -MP

# Internal C flags that should not be changed by the user.
override KCFLAGS += \
    -std=gnu11 \
    $(SHARED_FLAGS)

# Internal C++ flags that should not be changed by the user.
override KCXXFLAGS += \
    -std=gnu++20 \
    -fno-rtti \
    -fno-exceptions \
    $(SHARED_FLAGS)

# Internal nasm flags that should not be changed by the user.
override KNASMFLAGS += \
    -Wall \
    -f elf64

# Internal linker flags that should not be changed by the user.
override KLDFLAGS += \
    -m elf_x86_64 \
    -nostdlib \
    -static \
    -z max-page-size=0x1000 \
    -gc-sections \
    -T linker.ld

# Use "find" to glob all *.c, *.cpp, *.S, and *.asm files in the tree and obtain the
# object and header dependency file names.
override CFILES := $(shell cd src && find -L * -type f -name '*.c' | LC_ALL=C sort)
override CXXFILES := $(shell cd src && find -L * -type f -name '*.cpp' | LC_ALL=C sort)
override ASFILES := $(shell cd src && find -L * -type f -name '*.S' | LC_ALL=C sort)
override NASMFILES := $(shell cd src && find -L * -type f -name '*.asm' | LC_ALL=C sort)
override OBJ := $(addprefix obj/,$(CFILES:.c=.c.o) $(CXXFILES:.cpp=.cpp.o) $(ASFILES:.S=.S.o) $(NASMFILES:.asm=.asm.o))
override HEADER_DEPS := $(addprefix obj/,$(CFILES:.c=.c.d) $(CXXFILES:.cpp=.cpp.d) $(ASFILES:.S=.S.d))

# Default target.
.PHONY: all
all: bin/$(OUTPUT)

# Link rules for the final executable.
bin/$(OUTPUT): GNUmakefile linker.ld $(OBJ)
	mkdir -p "$$(dirname $@)"
	$(KLD) $(OBJ) $(KLDFLAGS) -o $@

# Include header dependencies.
-include $(HEADER_DEPS)

# Compilation rules for *.c files.
obj/%.c.o: src/%.c GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(KCC) $(KCFLAGS) $(KCPPFLAGS_C) -c $< -o $@

# Compilation rules for *.cpp files.
obj/%.cpp.o: src/%.cpp GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(KCXX) $(KCXXFLAGS) $(KCPPFLAGS_CXX) -c $< -o $@

# Compilation rules for *.S files.
obj/%.S.o: src/%.S GNUmakefile
	mkdir -p "$$(dirname $@)"
	$(KCC) $(KCFLAGS) $(KCPPFLAGS_C) -c $< -o $@

# Compilation rules for *.asm (nasm) files.
obj/%.asm.o: src/%.asm GNUmakefile
	mkdir -p "$$(dirname $@)"
	nasm $(KNASMFLAGS) $< -o $@

# Remove object files and the final executable.
.PHONY: clean
clean:
	rm -rf bin obj

# Remove everything built and generated including downloaded dependencies.
.PHONY: distclean
distclean: clean
	rm -rf freestnd-c-hdrs freestnd-cxx-hdrs src/cc-runtime.c src/limine.h

# Install the final built executable to its final on-root location.
.PHONY: install
install: all
	install -d "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
	install -m 644 bin/$(OUTPUT) "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/"

# Try to undo whatever the "install" target did.
.PHONY: uninstall
uninstall:
	rm -f "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT)"
	-rmdir "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
