# Temp&Hum 17 Click
Rjesenje projektnog zadata iz Ugradjenih sistema

# Preduslovi za izradu projekta
  - Buildroot okruzenje konfigurisano za rad sa Linux 6.6 kernelom
  - SD kartica od minimalno 4GB
  - DE1-SoC ploca
>[!NOTE]
> Posto Buildroot konfiguracija sa laboratorijskih vjezbi nije kompatibilna za rad sa HS3001 senzorom, polazna Buildroot konfiguracija mora biti obezbijediti rad sa Linux 6.6 kernelom, konfiguracija ce detaljnije biti opisana u nastavku teksta.

# Konfiguracija Buildroot-a i prilagodjavanje SPL-a
Prije bilo cega moramo da prilagodimo SPL za izvrsavanje, to radimo podesavanjem opcije ***Bootloaders -> Uboot -> custom U-Boot patches*** u buildroot konfiguraciji i postavljamo ovaj parametar na vrijednost  **board/terasic/de1soc_cyclone5/patch/de1-soc-handoff.patch**. Takodje vrsimo i editovanje **boot-env.txt** fajla koji se nalazi u  **board/terasic/de1soc_cyclone5** folderu.

Nakon ovih koraka vrsimo konfiguraciju Buildroot okruzenja pomocu komande 
```
make menuconfig
```
Posto buildroot konfiguracija sa laboratorijskih vjezbi nije prilagodjena za rad sa HS3001 senzorom, u Buildroot konfiguraciji treba napraviti sljedece izmjene:

  U okviru **Toolchain**:
   - postaviti Toolchain type opciju na **_Buildroot toolchain_**
   - postaviti Kernel Headers opciju na **_Linux 6.6.x kernel headers_**

  U okviru **Kernel**:
   - postaviti Custom repository version opciju na ***sofpga-6.6***

Nakon ovih izmjena potrebno je pristupiti Buildroot Linux kernel konfiguraciji pomocu komande
```
make linux-menuconfig
```
Potrebno je ukljciti/iskljuciti sledece opcije za potrebe projekta:

  U okviru **Device Drivers -> Character Devices**:
   - iskljciti opciju ***NEWHAVEN LCD***

  U okviru **Device Drivers -> Hardware Monitoring Support**:
   - ukljuciti opciju ***Renesas HS3001 humidity and temperature sensors*** ( kao modul <M> )

# Prilagodjavanje DTS fajla
Nakon sto smo konfigurisali Buildroot okruzenje, potrebno je da podesimo nas **socfpga_cyclone5_de1_soc.dts** fajl da radi sa HS3001 senzorom. U okviru fajla prije svega moramo osposobiti ***&i2c2*** bus kreiranjem novog cvora pod istoimenim nazivom. Unutar cvora ukljucujemo podcvor za temp&hum senzor, koji ce imati compatible string ***"renesas,hs3001"***, i **req = <0x44>** iz razloga sto je adresa temp&hum 17 click 0x44. Cvor treba da ima sljedeci oblik:
```
&i2c2 { 
    status = "okay";
	clock-frequency = <100000>;

    tempHum: tempHum@44 {
        status = "okay";
		compatible = "renesas,hs3001";
		reg = <0x44>;
	};
};
```
Nakon podesavanja .dts fajla, sve je spremno za kreiranje slike za SD karticu, te koristimo komandu:
```
make clean
make
```
Nakon ovoga neophodno je da se prebacimo u folder output/images, te da prebacimo sdcard.img fajl na nasu SD karticu pomocu dd komande : 
```
sudo dd if=sdcard.img of=/dev/sdb bs=1M
```
Zadnji korak je prebacivanje ***socfpga.rbf*** fajla na **FAT32** particiju nase kartice, te nakon toga nase okruzenje je spremno za izvrsavanje i rad sa HS3001 senzorom.


# Povezivanje Temp&Hum 17 Click ploce na DE1-SoC plocu

![image](https://github.com/user-attachments/assets/438289c6-b238-4a5b-be65-67dc97dd048b)

Kao sto vidimo na slici, Temp&Hum 17 Click ploca se povezuje na odgovarajuce pinove **GPIO0** konektora. Pored **SCL** i **SDA** pinova, moramo povezati uzemljenje i napajanje(3.3V). Oni se spajaju na **GPIO1** kontroleru. 


![Connection (1)](https://github.com/user-attachments/assets/1d5275e7-23d8-4521-8200-7299f6f8a3e4)

Po povezivanju i pokretanju minicom programa, prvo sto moramo uraditi je ucitati drajver za hs3001 senzor pomocu komande:

```
modprobe hs3001
```

nakon toga mozemo provjeriti da li je uredjaj registrovan na odgovarajucu adresu ( 0x44 ) pomocu komande:

```
i2cdetect -ry 2
```

Nakon toga mozemo pokusati pristupiti vrijednostima temperature ocitanim pomocu HS3001 senzora iscitavanjem fajla: 

```
/sys/class/hwmon/hwmon0/temp1_input
```
i vlaznosti iscitavanjem fajla:

```
/sys/class/hwmon/hwmon0/hum1_input
```

![370687590-0bc984a8-dd98-4176-815d-79f82d0e122d](https://github.com/user-attachments/assets/b468bb64-2d16-4bcf-bde0-82b43c21edbd)


Ali kao sto vidimo, iscitavanje ovih fajlova dovodi do greske "no zero length". O cemu se radi? Problem je sto nasa razvojna ploca, DE1-SoC ne podrzava **zero-length write** operacije, sto mozemo vidjeti ako pogledamo **i2c-core-base.c** fajl: 

```
for (i = 0; i < num; i++) {
        u16 len = msgs[i].len;

        if (msgs[i].flags & I2C_M_RD) {
            if (do_len_check && i2c_quirk_exceeded(len, q->max_read_len))
                return i2c_quirk_error(adap, &msgs[i], "msg too long");

            if (q->flags & I2C_AQ_NO_ZERO_LEN_READ && len == 0)
                return i2c_quirk_error(adap, &msgs[i], "no zero length");
        } else {
            if (do_len_check && i2c_quirk_exceeded(len, q->max_write_len))
                return i2c_quirk_error(adap, &msgs[i], "msg too long");

            if (q->flags & I2C_AQ_NO_ZERO_LEN_WRITE && len == 0)
                return i2c_quirk_error(adap, &msgs[i], "no zero length");
        }
    }
```
Dalje, u HS3001 drajveru (hs3001.c) mozemo naci sljedecu funkciju:

```
static int hs3001_read(struct device *dev, enum hwmon_sensor_types type,
               u32 attr, int channel, long *val)
 unutar koje se poziva ova funkcija: 

i2c_master_send(client, NULL, 0);
```
Zakljucak je da ova funkcija salje **zero-length write** operaciju koja nije podrzana na DE1-SoC ploci, te iz tog razloga dobijamo error i ne mozemo da procitamo vrijednosti temperature i vlaznosti vazduha. 

# Kako rijesiti ovaj problem? 
Preporuka je da se koristi ploca koja ce imati podrsku za ovu vrstu upisa, poput Raspberry pi ploce, koja ima omogucen **zero-length write** operaciju.	
