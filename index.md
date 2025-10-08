# myo-gesture-control-armband

Guide til Myo, på Windows computere (labbet har måske nogle der kan lånes :)

<img src="https://i.ytimg.com/vi/te1RBQQlHz4/maxresdefault.jpg" alt="Myo armbånd" width="400">

Denne guide er taget næsten 1:1 fra [interaction-lab.wiki.utwente.nl/playground:myo:myo_python](https://interaction-lab.wiki.utwente.nl/playground:myo:myo_python), men med ekstra kode og præcisering.

## Vigtigt
- Myos hjemmeside og officiel support findes ikke mere, så det er baseret på community guides og repositories.
- Myo virker ikke på nye MacBooks (silicon chips: M1, M2, M3, M4 osv.).
- Der er ikke officiel support til Linux/Raspberry Pi, men måske er der guides til det på nettet – tror dog det er svært :D
- Altså er denne guide primært til Windows PC, men det skulle også gerne virke på Intel MacBooks (har kun testet på Windows).
- På Windows virker det rigtig godt.
- hvis man bare vil lege med myos'eget software, som kan sjove ting såsom at styre slideshows, og styre computermusen, kan man nøjes med at bruge Myo Connect appen ([download .exe-filen herfra (hvis du bruger windows)](https://github.com/NiklasRosenstein/myo-python/releases)) kør appen og følg guiden. Så behøves man ikke at downloade andet og skrive kode.


## Step-by-step setup

1. **Download Myo SDK**  
    Download og gem Myo SDK et sted (fx Documents eller Desktop).  
    [Download her](https://github.com/NiklasRosenstein/myo-python/releases)

2. **Tilføj SDK til systemets PATH**  
    For Windows:  
    - Åbn **Edit the system environment variables**
    - Naviger til **Environment variables**
    - I det øverste vindue, vælg **Path** og tryk **Edit**
    - Vælg **Browse** og naviger til mappen `...\myo-sdk-win-0.9.0\bin` (hvor du har gemt og udpakket SDK)
    - Genstart din PC

3. **Verificer at armbåndet er tilsluttet**  
    Brug Myo Connect appen ([download .exe-filen herfra (hvis du bruger windows)](https://github.com/NiklasRosenstein/myo-python/releases)), kør appen og følg guiden i appen.

4. **Installer nødvendigt Python-bibliotek**  
    Åbn terminalen og kør:
    ```sh
    pip install "myo-python>=1.0.0"
    ```
    Sørg for at computeren har Python installeret først.

    ## Kode 1: Test om Myo virker

    Her er et eksempel på Python-kode, der tester om armbåndet er tilsluttet og reagerer:

    ```python
    import myo

    class Listener(myo.DeviceListener):
        def on_paired(self, event):
            print(f"Hello, {event.device_name}!")
            event.device.vibrate(myo.VibrationType.short)

        def on_unpaired(self, event):
            print("No paired device present")
            return False  # Stop hub

    if __name__ == '__main__':
        
        myo.init(sdk_path=r'C/Users/ddlab/Documents/myo-sdk-win-0.9.0') # VIGTIGT: juster path alt efter hvor i har gemt sdk mappen henne
        hub = myo.Hub()
        listener = Listener()
        while hub.run(listener.on_event, 500):
            pass
    ```

    **Bemærk:**  
    - Husk at rette `sdk_path` til den korrekte sti, hvor du har udpakket Myo SDK.
    - Koden vibrerer armbåndet kort, når det er tilsluttet.
    - Hvis armbåndet ikke er tilsluttet, stopper programmet.
    - Python-biblioteket `myo-python` skal være installeret.

    ## Kode 2: Pose-detektion og udskrivning af gestur-nummer

    Denne kode opdager hvilken gestur (pose) der bruges og printer et tal for hver gestur:

    - `0` = afslappet hånd  
    - `1` = knytnæve ("Fist")  
    - `2` = fingers spredt ("Fingers spread")  
    - `3` = bøjet hånd ("Wave in")  
    - `4` = strakt hånd ("Wave out")  
    - `5` = tommelfinger rører langemand ("Double tap")  

    Det er altså nemt herfra og tilføje et if-statement, der gør noget alt efter hvilket tal/pose der registreres

    ```python
    import myo

    class PoseLogger(myo.DeviceListener):
        def on_connected(self, event):
            print(f"Connected to {event.device_name}")
            event.device.stream_emg(False)  # Disable EMG if you don’t need it

        def on_pose(self, event):
            print(f"Pose detected: {event.pose}")

    if __name__ == '__main__':
        # Initialize Myo SDK (replace with correct path if needed)
        myo.init(sdk_path=r'C:\Users\ddlab\Documents\myo-sdk-win-0.9.0')

        hub = myo.Hub()
        listener = PoseLogger()

        try:
            print("Listening for poses... (Ctrl+C to quit)")
            while hub.run(listener.on_event, 500):
                pass
        except KeyboardInterrupt:
            print("Exiting...")
        finally:
            hub.stop()
    ```


    **Bemærk:**  
    - Husk at rette `sdk_path` til den korrekte sti, hvor du har udpakket Myo SDK.
    - Koden printer både gestur-navn og tilhørende nummer.
    - Gestur-numrene kan tilpasses efter behov.
    - Koden kræver at `myo-python` er installeret.


    ## Kode 3: Data logging til .csv (avanceret)

    Læs og plot armbåndets værdier  
    Dette script forbinder til armbåndet, venter på at posen "Fist" registreres, og begynder derefter at læse og gemme data fra armbåndet i en CSV-fil. Når "Fingers spread" registreres, gemmes data, et plot vises, og programmet afsluttes.

    Formålet med koden er at vise, hvordan man bruger forskellige `on_something`-funktioner til at optage data, samt hvordan pose-detektion for standard-poser kan implementeres i Python.

    ```python
    # Imports for tid, myo, tidsinterval, lagring og plot af data
    import myo
    from myo.utils import TimeInterval
    import csv
    import matplotlib.pyplot as plt

    class Listener(myo.DeviceListener):
        def __init__(self):
            self.interval = TimeInterval(None, 0.05)  # tid i sekunder
            self.orientation_data = []
            self.emg_data = []
            self.rssi = None

        def on_connected(self, event):
            event.device.request_rssi()
            print(f"Hello, {event.device_name}!")
            event.device.stream_emg(True)

        def on_rssi(self, event):
            self.rssi = event.rssi
            print(f"received signal strength = {self.rssi}")

        def on_emg(self, event):
            if not self.interval.check_and_reset():
                return
            self.emg_data.append(event.emg)

        def on_orientation(self, event):
            if not self.interval.check_and_reset():
                return
            self.orientation_data.append(event.orientation)

        def save_emg_to_csv(self, filename):
            with open(filename, 'w', newline='') as csvfile:
                writer = csv.writer(csvfile)
                writer.writerows(self.emg_data)
            print(f'Data saved to {filename}.')

        def save_orientation_to_csv(self, filename):
            with open(filename, 'w', newline='') as csvfile:
                fieldnames = ['w', 'x', 'y', 'z']
                writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
                writer.writeheader()
                for row in self.orientation_data:
                    writer.writerow({'w': row[0], 'x': row[1], 'y': row[2], 'z': row[3]})
            print(f'Data saved to {filename}.')

        def plot_emg_data(self):
            fig, axs = plt.subplots(8, 1, figsize=(10, 20), sharex=True)
            fig.suptitle('EMG Data')
            for i in range(8):
                axs[i].plot([sample[i] for sample in self.emg_data])
                axs[i].set_ylabel(f'Channel {i + 1}')
            plt.tight_layout()
            plt.subplots_adjust(top=0.95)
            plt.show()

    if __name__ == '__main__':
        myo.init(sdk_path=r'C/Users/ddlab/Documents/myo-sdk-win-0.9.0') # VIGTIGT: juster path alt efter hvor i har gemt sdk mappen henne
        hub = myo.Hub()
        listener = Listener()
        try:
            print("Collecting data. Please perform gestures...")
            while hub.run(listener.on_event, 500):
                pass
        except KeyboardInterrupt:
            pass
        finally:
            hub.stop()
            listener.save_emg_to_csv('csv/myo_emg_data.csv')
            listener.save_orientation_to_csv('csv/myo_orient_data.csv')
            listener.plot_emg_data()
    ```

    **Bemærk:**  
    - Husk at rette `sdk_path` til den korrekte sti, hvor du har udpakket Myo SDK.
    - Husk at oprette mappen `csv/` før du kører koden, ellers får du fejl.
    - Koden gemmer både EMG- og orienteringsdata i separate CSV-filer.
    - Plotter EMG-data for alle 8 kanaler.
    - Du kan tilpasse koden til at reagere på specifikke poses, fx "Fist" og "Fingers spread".
    - Kræver at `matplotlib` er installeret (`pip install matplotlib`).
    - Koden er avanceret og kræver forståelse for Python og Myo SDK.
