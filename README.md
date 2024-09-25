# Temp&Hum 17 Click
Rjesenje projektnog zadata iz Ugradjenih sistema

# Preduslovi za izradu projekta
  - Buildroot okruzenje konfigurisano za rad sa Linux 6.6 kernelom
  - SD kartica od minimalno 4GB
  - DE1-SoC ploca
Posto Buildroot konfiguracija sa laboratorijskih vjezbi nije kompatibilna za rad sa HS3001 senzorom, polazna Buildroot konfiguracija mora biti obezbijediti rad sa Linux 6.6 kernelom, konfiguracija ce detaljnije biti opisana u nastavku teksta.

# Konfiguracija Buildroot konfiguracije i prilagodjavanje SPL-a
Prije bilo cega moramo da prilagodimo SPL za izvrsavanje, to radimo podesavanjem opcije bootloaders -> uboot -> custom U-Boot patches u buildroot konfiguraciji i postavljamo ovaj parametar na vrijednost  board/terasic/de1soc_cyclone5/patch/de1-soc-handoff.patch. Takodje vrsimo i editovanje boot-env.txt fajla koji se nalazi u  board/terasic/de1soc_cyclone5 folderu.

Nakon ovih koraka vrsimo konfiguraciju Buildroot okruzenja pomocu komande make menuconfig. Posto buildroot konfiguracija sa laboratorijskih vjezbi nije prilagodjena za rad sa HS3001 senzorom, u Buildroot konfiguraciji treba napraviti sljedece izmjene:
  U okviru Toolchain:
   - postaviti Toolchain type opciju na Buildroot toolchain
   - postaviti Kernel Headers opciju na Linux 6.6.x kernel headers
  U okviru Kernel:
   - postaviti Custom repository version opciju na sofpga-6.6
Nakon ovih izmjena potrebno je pristupiti Buildroot Linux kernel konfiguraciji pomocu komande make linux-menuconfig. Potrebno je ukljciti/iskljuciti sledece opcije za potrebe projekta:
  U okviru Device Drivers -> Character Devices:
   - iskljciti opciju NEWHAVEN LCD
  U okviru Device Drivers -> Hardware Monitoring Support:
   - ukljuciti opciju Renesas HS3001 humidity and temperature sensors ( kao modul <M> )
# Prilagodjavanje DTS fajla
Nakon sto smo konfigurisali Buildroot okruzenje, potrebno je da podesimo nas socfpga_cyclone5_de1_soc.dts fajl da radi sa HS3001 senzorom. U okviru fajla prije svega moramo osposobiti &i2c2 bus kreiranjem novog cvora pod istoimenim nazivom. Unutar cvora ukljucujemo podcvor za temp&hum senzor, koji ce imati compatible string "renesas,hs3001", i req = <0x44> iz razloga sto je adresa temp&hum 17 click 0x44. Cvor treba da ima sljedeci oblik:
&i2c2 { 
    status = "okay";
	clock-frequency = <100000>;

    tempHum: tempHum@44 {
        status = "okay";
		compatible = "renesas,hs3001";
		reg = <0x44>;
	};
};
Nakon podesavanja .dts fajla, sve je spremno za kreiranje slike za SD karticu, te koristimo komandu:

make clean
make

Nakon ovoga neophodno je da se prebacimo u folder output/images, te da prebacimo sdcard.img fajl na nasu SD karticu pomocu dd komande : 

sudo dd if=sdcard.img of=/dev/sdb bs=1M

Zadnji korak je prebacivanje socfpga.rbf fajla na FAT32 particiju nase kartice, te nakon toga nase okruzenje je spremno za izvrsavanje i rad sa HS3001 senzoro.

#Povezivanje Temp&Hum 17 Click ploce na DE1-SoC plocu

