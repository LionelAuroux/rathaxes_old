Copy of the archive on code.google

Rathaxes, a DSL for driver developement

# rathaxes

Rathaxes is a DSL (domain specific language) which will allow to describe the driver completely.

## motivations

According to recent studies, up to 70% of an operating system's code is driver code. This code is often quite technical, very close to hardware, and up to 7 times more bug-prone than normal code. Rathaxes allow you to describe easily interactions between the kernel and low level registries.
sample of a serial driver

```
DEVICE rs232
{
    
    REGISTER(R) BIT[8] rcv_buff LIKE(........) @(ioport + 0);
    
    REGISTER(W) BIT[8] snd_buff LIKE(........) @(ioport + 0);
    
    REGISTER(RW) BIT[8] ier LIKE (**......)
        @(resources::ioport + 1)
        {
            [0] AS data_available;
            [1] AS transmitter_empty;
            [2] AS line_status_change;
            [3] AS modem_status_change;
            [4] AS sleep_mode;
            [5] AS low_power_mode;
        };
    
    REGISTER(RW) BIT[8] lcr LIKE (........) @(ioport + 3)
        {
            [0..1] AS word_lengh
            {
                (00) -> _5bits;
                (01) -> _6bits;
                (10) -> _7bits;
                (11) -> _8bits;
            };
    
            [2] AS stop_bits
            {
                (0) -> _1stop_bits;
                (1) -> _2stop_bits;
            };
    
            [3..5] AS parity_type
            {
                (000) -> none;
                (001) -> odd;
                (011) -> even;
                (101) -> high;
                (111) -> low;
            };
    
            [6] AS break_signal
            {
                (0) -> disable;
                (1) -> enable;
            };
    
            [7] AS dlab
            {
                (0) -> buffers;
                (1) -> clock;
            };
        };
    
    //Modem Control
    REGISTER(RW) BIT[8] mcr LIKE (****.*..) @(ioport + 4)
        {
            [0] AS dtr; // Data Terminal Ready
            [1] AS rts; // Request To Send
            [3] AS ao2; // Auxiliary Output 2
        };
    
    REGISTER(RW) BIT[8] dll LIKE(........) @(ioport + 0);
    
    REGISTER(RW) BIT[8] dlm LIKE(........) @(ioport + 1);
    
    REGISTER(R) BIT[8] lsr LIKE(........) @(ioport + 5)
        {
            [0] AS data_available
            {
                (0) -> FALSE;
                (1) -> TRUE;
            };
    
            // customised name for the bits state
            [1] AS overrun
            {
                (0) -> good;
                (1) -> error;
            };
    
            [2] AS parity
            {
                (0) -> good;
                (1) -> error;
            };
    
            [3] AS framing
            {
                (0) -> good;
                (1) -> error;
            };
    
            [4] AS break_signal
            {
                (0) -> TRUE;
                (1) -> FALSE;
            };
    
            [5] AS thr_state
            {
                (0) -> transmitting;
                (1) -> empty;
            };
    
            [6] AS thr_and_line
            {
                (0) -> transmitting;
                (1) -> empty_idle;
            };
    
            [7] AS data_fifo
            {
                (0) -> good;
                (1) -> error;
            };
        };
    
    PUBLIC PROPERTY WORD divisor
    {
        SET
        {
            SET(lcr.dlab, 1);
            SET(dll, value[0..7]);
            SET(dlm, value[8..15]);
            SET(lcr.dlab, 0);
        }
    
        GET
        {
            SET(lcr.dlab, 1);
            SET(value[0..7], dlm);
            SET(value[8..15], dll);
            SET(lcr.dlab, 0);
        }
    };
    
};

KERNEL_INTERFACES interface_rs232
{
    read(CONTEXT context, BUFFER output)
    {
        LOG("my_rs232 [read]\n");
        CONCAT(output, rcv_buff) PRE { WAIT(lsr.data_available, lsr.data_available->TRUE); }; 
    };

    write(CONTEXT context, BUFFER input)
    {
        LOG("my_rs232 [write]\\n");
        WAIT(lsr.thr_and_line, lsr.thr_and_line->empty_idle);
        COPY(snd_buff, input)
            PRE
            {
                WAIT(lsr.thr_state, lsr.thr_state->empty);
            };
    };
    
    on_plug(CONTEXT context)
    {
        LOG("my_rs232 [on_plug]\\n");
        SET(divisor, 0x0c);
    
        // Set mode 8 data bits, 1 stop bit, no parity
        SET(lcr.word_lengh, lcr.word_lengh->_8bits);
        SET(lcr.stop_bits, lcr.stop_bits->_1stop_bits);
        SET(lcr.parity_type, lcr.parity_type->none);
    
        SET(mcr.rts, 1);
        SET(mcr.dtr, 1);
    };
    
    open(CONTEXT context)
    {
        LOG("my_rs232 [open]\\n");
    };
    
    close(CONTEXT context)
    {
        LOG("my_rs232 [close]\\n");
    };
    
};

SEQUENCES seq_rs232 
{
    my_sequence(CONTEXT context, DWORD in, BUFFER out, BIT fail)
    { LOG("my_rs232 [my_sequence]\n"); };
};

DRIVER my_rs232 { DEVICES = rs232; };

CONFIGURATION
{
    
    ARCH = x86; ioport = 0x03f8;
    
    OS linux
    {
        MAJOR = -1;
        TYPE = chardev;
        ioctl_magicnumber = 0xf5;
        VERSION = "2.6.18";
    };
    
    OS windows
    {
        VERSION = "Windows";
        DEVICE_NAME = GUID_DEVINTERFACE_RATHAXES_SERIAL;
        DEVICE_GUID = "1ADB9D41-6FE5-432e-A5FF-9B6911A150A4";
        CLASS_NAME = GUID_DEVCLASS_RATHAXES;
        CLASS_GUID = "E20808D9-5E08-4a2a-80A1-84DA0B2180AB";
        DISPATCH_MODE = WdfIoQueueDispatchParallel;
    };
    
    OS openBSD
    {
        MAJOR = -1;
        TYPE = chardev;
        ioctl_magicnumber = 0xf5;
        VERSION = "4.3";
    };
   
};

```

