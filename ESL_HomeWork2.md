# ESL_HomeWork2

### Information

---

**110061637  林佳瑩**

**Github link** **:** [https://github.com/nthu110061637/ESL_HomeWork2](https://github.com/nthu110061637/ESL_HomeWork2)

### Introduction

---

**Gaussian Blur with a TLM-2.0 interconnect**

- Testbench will read/write  pixels through the simple bus module from and to the Gaussian filter.
- The transaction is sent forward from initiator to target on the bus.

### Implementation

---

- Architecture
    
    ![Untitled](https://raw.githubusercontent.com/nthu110061637/ESL_HomeWork2/main/Architecture.jpg)
    
- main.cpp
    - Instantiate a bus with one target socket and one initiator socket.
        
        ```cpp
        SimpleBus<1, 1> bus("bus"); 
        ```
        
    - Initiator socket is bind to target socket at bus side.
        
        ```cpp
        tb.initiator.i_skt(bus.t_skt[0]); 
        ```
        
    - Initiator socket at bus side is bind to target socket at Gaussian filter side.
        
        ```cpp
        bus.i_skt[0](gaussian_filter.t_skt);
        ```
        
    - Setup the global memory map address "Gaussian_MM_BASE" to a port ID "0".
        
        It means that any blocking transport calls to the address will be sent to initiator socket with port ID "0".
        
        ```cpp
        bus.setDecode(0, Gaussian_MM_BASE, Gaussian_MM_BASE + Gaussian_MM_SIZE - 1);
        ```
        
- GaussianFilter.cpp
    - In the constructor we instantiate the t_skt for target socket at filter side.
    - Target socket will bind "blocking_transport" function call into the function pointer.
    
    ```cpp
    	GaussianFilter::GaussianFilter(sc_module_name n)
        : sc_module(n), t_skt("t_skt"), base_offset(0) {
      SC_THREAD(do_filter);
    
      t_skt.register_b_transport(this, &GaussianFilter::blocking_transport);
    	}
    ```
    
    - The blocking transport function
    
    ```cpp
    	/* GaussianFilter.cpp */
    	void GaussianFilter::blocking_transport(tlm::tlm_generic_payload &payload,
                                         sc_core::sc_time &delay) {
      sc_dt::uint64 addr = payload.get_address();
      addr -= base_offset;
      unsigned char *mask_ptr = payload.get_byte_enable_ptr();
      unsigned char *data_ptr = payload.get_data_ptr();
      word buffer;
      switch (payload.get_command()) {
    	/* We can know initiator want to read from or write to the socket. */
      case tlm::TLM_READ_COMMAND:
        switch (addr) {
        case Gaussian_FILTER_RESULT_ADDR:
    			/* read from o_result fifo to buffer */
          buffer.uint = o_result.read();
          break;
        case Gaussian_FILTER_CHECK_ADDR:
          buffer.uint = o_result.num_available();
    			/* check if the o_result fifo is available or not */
        break;
        default:
          std::cerr << "Error! GaussianFilter::blocking_transport: address 0x"
                    << std::setfill('0') << std::setw(8) << std::hex << addr
                    << std::dec << " is not valid" << std::endl;
        }
        data_ptr[0] = buffer.uc[0];
        data_ptr[1] = buffer.uc[1];
        data_ptr[2] = buffer.uc[2];
        data_ptr[3] = buffer.uc[3];
        break;
      case tlm::TLM_WRITE_COMMAND:
        switch (addr) {
        case Gaussian_FILTER_R_ADDR:
          if (mask_ptr[0] == 0xff) {
            i_r.write(data_ptr[0]);
          }
          if (mask_ptr[1] == 0xff) {
            i_g.write(data_ptr[1]);
          }
          if (mask_ptr[2] == 0xff) {
            i_b.write(data_ptr[2]);
          }
          break;
        default:
          std::cerr << "Error! GaussianFilter::blocking_transport: address 0x"
                    << std::setfill('0') << std::setw(8) << std::hex << addr
                    << std::dec << " is not valid" << std::endl;
        }
        break;
      }
    ```
    
- GaussianFilter.h
    - When blocking transport function receive one transaction then the fifo will help us to make that if there is something available in the fifo then the do_filter() will do the computation.
        
        ```cpp
        	sc_fifo<unsigned char> i_r;
          sc_fifo<unsigned char> i_g;
          sc_fifo<unsigned char> i_b;
          sc_fifo<int> o_result;
        ```
        
- filter_def.h
    - Make the variable can access in different way.
    
    ```cpp
    union word {
      int sint;
      unsigned int uint;
      unsigned char uc[4];
    };
    ```
    
    - We map each fifo to specific address in the transaction.
        
        So the address maps one action the blocking transport function will do according to the address get from payload.get_address()
        
    
    ```cpp
    const int Gaussian_FILTER_R_ADDR = 0x00000000;
    const int Gaussian_FILTER_RESULT_ADDR = 0x00000004;
    const int Gaussian_FILTER_CHECK_ADDR = 0x00000008;
    ```
    
- Testbench.cpp
    - Instantiate the initiator in the constructor
        
        ```cpp
        Testbench::Testbench(sc_module_name n)
            : sc_module(n), initiator("initiator"), output_rgb_raw_data_offset(54) {
          SC_THREAD(do_Gaussian);
        }
        ```
        
    - We only need 8 bytes for RGB. (i.e. data.uc[3] will not be used in target module)
        
        So, we need to send mask to make target side know that which data is meaningful.
        
    - And then the initiator send the write_to_socket transaction to the target to call the transport funciton that define in the GaussianFilter.cpp.
        
        ```cpp
        data.uc[0] = R;
        data.uc[1] = G;
        data.uc[2] = B;
        mask[0] = 0xff;
        mask[1] = 0xff;
        mask[2] = 0xff;
        mask[3] = 0;
        initiator.read_from_socket(Gaussian_MM_BASE + Gaussian_FILTER_CHECK_ADDR, mask, data.uc, 4);
        ```
        
- Initiator.cpp
    - Instantiate i_skt("i_skt") in the constructor.
    
    ```cpp
    Initiator::Initiator(sc_module_name n) : sc_module(n), i_skt("i_skt") {}
    ```
    
    - Define the generic payload object that defines what transaction should do.
    - i_skt is linked to the function which t_skt register.
    
    ```cpp
    	int Initiator::read_from_socket(unsigned long int addr, unsigned char mask[],
                                    unsigned char rdata[], int dataLen) {
      // Set up the payload fields. Assume everything is 4 bytes.
      trans.set_read();
      trans.set_address((sc_dt::uint64)addr);
    
      trans.set_byte_enable_length((const unsigned int)dataLen);
      trans.set_byte_enable_ptr((unsigned char *)mask);
    	/* describe which of the byte you send have meaning */
    
      trans.set_data_length((const unsigned int)dataLen);
      trans.set_data_ptr((unsigned char *)rdata);
    
      // Transport.
      do_trans(trans);
    
      /* For now just simple non-zero return code on error */
      return trans.is_response_ok() ? 0 : -1;
    
    } // read_from_socket()
    
    int Initiator::write_to_socket(unsigned long int addr, unsigned char mask[],
                                   unsigned char wdata[], int dataLen) {
      // Set up the payload fields. Assume everything is 4 bytes.
      trans.set_write();
      trans.set_address((sc_dt::uint64)addr);
    
      trans.set_byte_enable_length((const unsigned int)dataLen);
      trans.set_byte_enable_ptr((unsigned char *)mask);
    
      trans.set_data_length((const unsigned int)dataLen);
      trans.set_data_ptr((unsigned char *)wdata);
    
      // Transport.
      do_trans(trans);
    
      /* For now just simple non-zero return code on error */
      return trans.is_response_ok() ? 0 : -1;
    
    } // writeUpcall()
    
    void Initiator::do_trans(tlm::tlm_generic_payload &trans) {
      sc_core::sc_time dummyDelay = sc_core::SC_ZERO_TIME;
      i_skt->b_transport(trans, dummyDelay);
      //wait(sc_core::SC_ZERO_TIME);
      wait(dummyDelay);
    } 
    ```
    
- SimpleBus.h
    - Make different target socket to bind with different transport function.
    - But we only implement one target here by SimpleBus<1, 1> bus("bus").
    - So, we only register the b_transport to the target socket as port id “0”.
    
    ```cpp
    for (unsigned int i = 0; i < NR_OF_TARGET_SOCKETS; ++i) {
          t_skt[i].register_b_transport(this, &SimpleBus::initiatorBTransport, i);
          t_skt[i].register_transport_dbg(this, &SimpleBus::transportDebug, i);
          t_skt[i].register_get_direct_mem_ptr(this, &SimpleBus::getDMIPointer, i);
        }
    ```
    
    - initiatorBTransport model the interconnect delay.
    
    ```cpp
    t = t + delay(trans); 
    (*decodeSocket)->b_transport(trans, t);
    ```
    
- MemoryMap.h
    - Through the getMapping we can know if the address is corresponding port address region and then return decode.
    
    ```cpp
    icmPortMapping **decodes;
    
      icmPortMapping *getMapping(int port, Addr address) {
        icmPortMapping *decode;
        for (decode = decodes[port]; decode; decode = decode->getNext()) {
          if (decode->inRegion(address)) {
            return decode;
          }
        }
        return 0;
      }
    ```
    
    - getPortId is used to find the “id” of the targets and to calculate the offset in local memory and it will be modified directly because “offset” is a reference.
    
    ```cpp
    int getPortId(const Addr address, Addr &offset) {
        unsigned int i;
        for (i = 0; i < no_of_targets(); i++) {
          icmPortMapping *decode = getMapping(i, address);
          if (decode) {
            offset = decode->offsetInto(address);
            return i;
          }
        }
        return -1;
      }
    ```
    
    - serDecode will distribute the local memory space by *decode.
    
    ```cpp
    void setDecode(int portId, Addr lo, Addr hi) {
        if (portId >= static_cast<int>(no_of_targets())) {
          printf("Error configuring TLM decoder %s: portId (%d) cannot be greater "
                 "than the number of targets (%d)\n",
                 memory_map_name().c_str(), portId, no_of_targets());
          return;
          abort();
        }
        if (lo > hi) {
          printf("Error configuring TLM decoder %s: lo (0x%llX) cannot be greater "
                 "than hi (0x%llX)\n",
                 memory_map_name().c_str(), lo, hi);
          return;
        }
        icmPortMapping *decode = new icmPortMapping(lo, hi);
        decode->setNext(decodes[portId]);
        decodes[portId] = decode;
      }
    ```
    

### Result

---

- Before
    
    ![Untitled](https://github.com/nthu110061637/ESL_HomeWork1/blob/main/hw1_2/build/originrb.bmp)
    
- After
    
    ![Untitled](https://raw.githubusercontent.com/nthu110061637/ESL_HomeWork2/main/hw2/build/out.bmp)
