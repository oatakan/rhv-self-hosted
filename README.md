# rhv-self-hosted

GPU passthrough:

add the following custom parameter on the VM:

["-enable-kvm","-cpu","host,kvm=off,hv_time,hv_relaxed,hv_vapic,hv_spinlocks=0x1fff,hv_vendor_id=Nvidia43FIX","-drive","if=pflash,format=raw,readonly,file=/usr/share/edk2.git/ovmf-x64/OVMF_CODE-pure-efi.fd"]