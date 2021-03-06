Nordic's nRF91 series has a mechanism for storing [TLS credentials](https://infocenter.nordicsemi.com/index.jsp?topic=%2Fref_at_commands%2FREF%2Fat_commands%2Fmob_termination_ctrl_status%2Fcmng_set.html) securely on the "modem side" of the SoC. The purpose of this project is to provide a proof-of-concept process for writing these credentials efficiently -- using only the SWD interface and without invoking a compiler.
### About
There are several reasons why compiling TLS credentials into production firmware is **not** a good idea:
* The application must copy the credentials to the modem side of the SoC so they end up occupying space on both cores
  * Individual certificates can approach 4KB in length
  * This also means that the application has to contain extra code to perform the copying
* Key material that is part of the application doesn't benefit from all of the extra security that is provided by the modem core
* Compiling credentials into application hex files requires generating unique application hex files for every device 

These disadvantages are fine during development but ideally a production workstation would require only the current version of the application firmware, credentials to use for the SoC, and an SWD interface for programming.

This project consists of two components:
1. A prebuilt firmware hex file (compiled using the [nRF Connect SDK](http://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)) that is responsible for deactivating the modem, writing a list of credentials to the modem side, and writing the device's IMEI along with a result code to flash memory.
1. A Python command line interface that adds credentials to the prebuilt hex file, programs it to the device, allows it to run, reads the IMEI and result code from flash to verify that it completed successfully, and then erases it.

This two-step process allows all devices to be deployed with the same application hex file and uses Python to do the heavy lifting during production (instead of requiring a full toolchain with a compiler). The extra step to write the credentials should only add on the order of tens of seconds to the overall programming process and provides a method for the nRF91's IMEI to be acquired.
### Requirements
The **intelhex** module is used for working with the hex files and the excellent **pynrfjprog** is used to program the SoC. Requirements can be installed from the command line using pip:
```
$ cd cred
$ pip3 install --user -r requirements.txt
```
### Usage
The command line interface can be modified to add additional capabilties. The existing functionality is pretty comprehensive:
```
$ python3 cred.py --help
usage: cred [-h] [-i IN_FILE_PATH] [-o OUT_FILE_PATH] [-d FW_EXECUTE_DELAY]
            [-s JLINK_SERIAL_NUMBER] [--sec_tag SEC_TAG] [--psk PRESHARED_KEY]
            [--psk_ident PRESHARED_KEY_IDENTITY] [--CA_cert CA_ROOT_CERT_PATH]
            [--client_cert CLIENT_CERT_PATH]
            [--client_private_key CLIENT_PRIVATE_KEY_PATH] [--imei_only]
            [--program_app APP_HEX_FILE_PATH]

A command line interface for managing nRF91 credentials via SWD.

optional arguments:
  -h, --help            show this help message and exit
  -i IN_FILE_PATH, --in_file IN_FILE_PATH
                        read existing hex file instead of generating a new one
  -o OUT_FILE_PATH, --out_file OUT_FILE_PATH
                        write output from read operation to file instead of
                        programming it
  -d FW_EXECUTE_DELAY, --fw_delay FW_EXECUTE_DELAY
                        delay in seconds to allow firmware on nRF91 to execute
  -s JLINK_SERIAL_NUMBER, --serial_number JLINK_SERIAL_NUMBER
                        serial number of J-Link
  --sec_tag SEC_TAG     sec_tag to use for credential
  --psk PRESHARED_KEY   add a preshared key (PSK) as a string
  --psk_ident PRESHARED_KEY_IDENTITY
                        add a preshared key (PSK) identity as a string
  --CA_cert CA_ROOT_CERT_PATH
                        path to a root Certificate Authority certificate
  --client_cert CLIENT_CERT_PATH
                        path to a client certificate
  --client_private_key CLIENT_PRIVATE_KEY_PATH
                        path to a client private key
  --imei_only           only read the IMEI and exit without writing any
                        credentials
  --program_app APP_HEX_FILE_PATH
                        program specified hex file to device before finishing

WARNING: nrf_cloud relies on credentials with sec_tag 16842753.
```
If only the IMEI is needed then no credentials have to be specified:
```
$ python3 cred.py --imei_only
123456789012345
```
The 15-digit IMEI is read from the device and written to stdout before the Python program exits.

A set of credentials that use the same sec_tag can be written to the SoC in a single step:
```
$ python3 cred.py --sec_tag 1234 --psk_ident nrf-123456789012345 --psk CAFEBABE
123456789012345
```
If PEM or CRT files are required then they are specified by file path instead of pasted onto the command line. If more than one sec_tag is required then they can be added by writing the first hex file to a file and then using that file as an input on successive iterations. Here the second invocation adds to the hex file from the first and then writes to the SoC:
```
$ python3 cred.py --sec_tag 1234 --psk_ident nrf-123456789012345 --psk CAFEBABE -o multi_cred.hex
$ python3 cred.py --sec_tag 3456 -i multi_cred.hex --CA_cert ca_file.crt
123456789012345
```
The Python program waits seven seconds after programming the hex file to allow it to process the credentials and then write a result code to a fixed location in the nRF91's flash memory. This result code is then read to verify that hex file had time to complete its task. If the default delay is not long enough then a longer value can be specified via the **--fw_delay** argument.

The prebuilt hex file can be modifed and compiled by moving this repo into the "ncs/nrf/samples/nrf9160/" directory and building it as usual. Checkout the appropriate tag for each NCS version e.g. NCSv1.2.0 for NCS v1.2.0 or v1.2.1.
### Limitations
The ability to add credentials to a file and then read from that file to add additional credentials on the next invocation is half-baked because credentials are not parsed and verified.

It may be necessary to recompile the hex file for custom PCBs. This hex file can be copied to the 'build' directory to replace the default one.
