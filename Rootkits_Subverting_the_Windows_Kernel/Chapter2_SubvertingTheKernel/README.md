Communication from user to kernel device and be done using ioctl

IRPs (IO Request Packets) contains buffer of data sent from anywhere

```C
NTSTATUS OnStubDispatch(IN PDEVICE_OBJECT DeviceObject,
                        IN PIRP Irp )
{
      Irp->IoStatus.Status = STATUS_SUCCESS;
      IoCompleteRequest(Irp,
                        IO_NO_INCREMENT );
      return STATUS_SUCCESS;
}
VOID OnUnload( IN PDRIVER_OBJECT DriverObject )
{
      DbgPrint("OnUnload called\n");
}
NTSTATUS DriverEntry( IN PDRIVER_OBJECT theDriverObject,
                      IN PUNICODE_STRING theRegistryPath )
{
      int i;
      theDriverObject->DriverUnload  = OnUnload;
      for(i=0;i< IRP_MJ_MAXIMUM_FUNCTION; i++ )
      {
            theDriverObject->MajorFunction[i] = OnStubDispatch;
      }
      return STATUS_SUCCESS;
}
```

User mode applications open the device as if were a file

```C
const WCHAR deviceNameBuffer[]  = L"\\Device\\MyDevice";
PDEVICE_OBJECT g_RootkitDevice; // Global pointer to our device object
NTSTATUS DriverEntry(IN PDRIVER_OBJECT  DriverObject,
                     IN PUNICODE_STRING RegistryPath )
{
    NTSTATUS                ntStatus;
    UNICODE_STRING          deviceNameUnicodeString;
    // Set up our name and symbolic link.
    RtlInitUnicodeString (&deviceNameUnicodeString,
                          deviceNameBuffer );
    // Set up the device.
    //
    ntStatus = IoCreateDevice ( DriverObject,
                                0, // For driver extension
                                &deviceNameUnicodeString,
                                0x00001234,
                                0,
                                TRUE,
                                &g_RootkitDevice );
    ...
```

```C
hDevice = CreateFile("\\\\Device\\MyDevice",
                     GENERIC_READ | GENERIC_WRITE,
                     0,
                     NULL,
                     OPEN_EXISTING,
                     FILE_ATTRIBUTE_NORMAL,
                     NULL
                     );
    if ( hDevice == ((HANDLE)-1) )
        return FALSE;
```

Soem rootkits use symbolic links. It's nice to have.

