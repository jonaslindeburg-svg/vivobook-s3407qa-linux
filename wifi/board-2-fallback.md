# WCN6855 board file for Vivobook S14 S3407QA

The kernel looks up:

    bus=pci,vendor=17cb,device=1103,subsystem-vendor=105b,subsystem-device=e130,
    qmi-chip-id=18,qmi-board-id=255,variant=AS_SA_S3407QA

No such entry exists in linux-firmware. The BDF in the Windows driver
(bdwlan_wcn685x_2p1_nfa765a_AS_SA_S3407QA.elf, 60000 bytes) is *unsigned* —
working entries are 60032 bytes, the extra 32 being a trailing signature that
cannot be recomputed. Using it directly gives MHI_CB_EE_RDDM and an unbootable
system.

Fix: keep the S3407QA lookup name, point it at an existing signed blob from a
sibling device (e0ce). Add to board-2.json:

    {
        "names": [
            "bus=pci,vendor=17cb,device=1103,subsystem-vendor=105b,subsystem-device=e130,qmi-chip-id=18,qmi-board-id=255,variant=AS_SA_S3407QA"
        ],
        "data": "bus=pci,vendor=17cb,device=1103,subsystem-vendor=105b,subsystem-device=e0ce,qmi-chip-id=18,qmi-board-id=255.bin"
    },

Note: names have NO suffix, data DOES end in .bin.

Build and install:

    zstd -d /usr/lib/firmware/ath11k/WCN6855/hw2.0/board-2.bin.zst
    ./ath11k-bdencoder -e board-2.bin        # extracts blobs + board-2.json
    # edit board-2.json, add the entry above
    ./ath11k-bdencoder -c board-2.json       # NOT "-c board-2"
    sudo install -Dm644 board-2-new.bin \
      /usr/lib/firmware/updates/ath11k/WCN6855/hw2.1/board-2.bin

hw2.1 files are symlinks to hw2.0 — extract from hw2.0, install to hw2.1.
The chip reports chip_id 0x12 (=18), board_id 0xff (=255).
Verified working: correct MAC from BDF, both bands scan.
Remaining harmless error: "failed to process regulatory info -22".

linux-firmware (Jul 2026) ships WCN6855/hw2.0/nfa765/ with amss.bin and
m3.bin but no board-2.bin, so this workaround is still required.
