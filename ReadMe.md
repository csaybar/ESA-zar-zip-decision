![657e3c5e0885ee4e5c2062c58f9aa094fa4b14a4](https://github.com/user-attachments/assets/8299e5b0-4251-455d-a507-5c3d55272735)

Important links:

- **[Product Structure and Format Definition — EOPF](https://cpm.pages.eopf.copernicus.eu/eopf-cpm/main/PSFDjan2024.html#product-structure-and-format-definition)**

- **[PANGEO discussion](https://discourse.pangeo.io/t/whats-the-best-file-format-to-chose-for-raster-imagery-and-masks-products/4555/)**


This is my personal opinion on the [PANGEO](https://discourse.pangeo.io/t/whats-the-best-file-format-to-chose-for-raster-imagery-and-masks-products/4555/) discussion regarding ESA's decision to use ZAR+ZIP format. Unfortunately, new users cannot post many links in the PANGEO discourse, so I decided to put it here.


# **Comment:**


I had no prior knowledge of this decision, and I’m quite surprised. COG+STAC is the standard format most of us use for **data dissemination**. 

So, why ZARR+ZIP? The reasoning stems from Section 4.3 of the [Copernicus EO Processing Framework](https://cpm.pages.eopf.copernicus.eu/eopf-cpm/main/PSFDjan2024.html#product-structure-and-format-definition). However, the comparison there is **misleading**, it pits ZARR against COG when the logical comparison should be ZARR+ZIP versus COG.

![TableFormatJustification](https://github.com/user-attachments/assets/903e5027-c2c5-4d51-8e4a-0dc8508d4533)

I want to clarify why I consider this comparison **misleading**. I'll focus on the yellow-highlighted cells for COG. I'm considering a **multiband COG**, the ZIP proposed by ESA Is zero compression ratio.

****Optimization for cloud environment: Legacy format upgrade****

Why is being a legacy of something considered bad here? A fair and logical comparison should **focus on the number of HTTP GET requests required to retrieve data from the server**.

*Typical Use Case Example*

Suppose a user wants to download **5 minicubes of 3×128×128 for a specific scene**. Each query is fully contained within a specific chunk.

1. **COG**
  * **2 GET Request** to read the file header and IFD.
  * **5 GET Requests** to fetch the continuous chunks of data.
  * **Total:** **7 GET operations** (n+2 GETs).
2. **ZARR (Consolidated)**
  * **1 GET Request** to read the consolidated metadata.
  * **5 GET Requests** to fetch the chunks.
  * **Total:** **6 GET operations** (n+1 GETs).
  * Additionally, this can work directly on servers without supporting Range Requests.
3. **ZARR+Zipped (ESA idea)**

  * First explore the ZIP structure:
  * **1 Range Request** for the End of Central Directory Record (EOCDR). [Check](https://en.wikipedia.org/wiki/ZIP_(file_format)#End_of_central_directory_record_(EOCD))
  * **1 Range Request** for the Central Directory File Header (CDFH). [Check](https://en.wikipedia.org/wiki/ZIP_(file_format)#Central_directory_file_header_(CDFH))
  * **1 Range Request** for the Local File Header of the consolidated metadata. [Check](https://en.wikipedia.org/wiki/ZIP_(file_format)#Local_file_header)
  * **1 Range Request** to read the consolidated metadata.
  * For each query:
    * **1 Range Request** for the Local File Header of the chunk.
    * **1 Range Request** for the chunk itself.
  * **Total:** **14 GET operations** (2n+4 GETs).

![ZIP-64 Internal Layout](https://upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/600px-ZIP-64_Internal_Layout.svg.png)
<p> ZIP-64 Internal Layout source: Wikipedia</p>



This is not a cherrypick; it highlights the reasons why ZARR+Zipped is terribly slow in cloud environment settings, as @RichardScottOZ pointed out. You may have also noticed that it is slow in Python. Zarr-Python uses zipfile, which is poorly optimized. Please check the official documentation for more details: https://docs.python.org/3/library/zipfile.html

> Decryption is extremely slow as it is implemented in native Python rather than C.

**Streaming: Read-only**

As mentioned earlier by @kirill.kzb, this claim is not correct. Many tools already support streaming for GeoTIFF, and GDAL itself has supported streaming operations for nearly a decade (https://lists.osgeo.org/pipermail/gdal-dev/2015-February/041094.html). You can find details in the GDAL documentation (https://gdal.org/en/stable/drivers/raster/gtiff.html#streaming-operations)


**Optimization for DataCube: NO**

If this means creating 4D datacubes, well it is quite true. Zarr excels here. 
The choice of format can significantly impact performance locally too:

- **GeoTIFF**: To build a 4D structure, from a GeoTIFF datalake, 
you must perform **two read operations per file** (see above) before accessing the data.
This results in a total of **(n + 2) * k** operations, where:
  - `n` = number of chunks,
  - `k` = number of files.

- **Zarr (Directory-Based)**: In contrast, Zarr's directory-based structure only requires 
**n * k + 1** operations. Since Zarr supports n-dimensional chunking, you can further 
reduce the number of operations by optimizing your chunking strategy.

- **Zarr (Zipped - ESA idea)**: However, the Zarr+ZIP format proposed by ESA, is less efficient. As
they are proposing to create a Zarr+ZIP for every "time acquisition" ([check](https://cpm.pages.eopf.copernicus.eu/eopf-cpm/main/PSFD/4-storage-formats.html#zarr-representation-of-eopf-data-products)).It will require a walk around the ZIP global and local headers, before accessing the data. It 
requires **(2n + 4) * k** operations.


**Zipped format for dissemination Source: Not available**

This claim is quite misleading. What does "not available" really mean? If I have 10 Cloud 
Optimized Geotiffs (COGs) and want to share them in a ZIP format, I can use GDAL's Virtual
File System (VFS) (https://gdal.org/en/latest/user/virtual_file_systems.html#vsizip-zip-archives).
Furthermore, it's possible to chain operations with GDAL VFS, as described in the same documentation
section (https://gdal.org/en/latest/user/virtual_file_systems.html#vsizip-zip-archives). This 
functionality will work on any system that supports the GDAL Raster API, which includes nearly 
all software in the Earth Observation (EO) community.

**Final thoughts**


Personally, I don’t know anyone who believes that ZARR+ZIP is a good choice for data dissemination. **I’m not sure how this conclusion was reached**. We love Zarr and please use it internally as a storage format if it's more convenient for your engineers, **but why would it be chosen as a dissemination format?**

Outside of Python, Zarr’s support is quite limited. **Zarr is not a CLEAR spatial format**. Yeah, I know it is a [community standard of the OGC](https://www.ogc.org/publications/standard/zarr-storage-specification/) but someone might call the coordinates “x, y,” another might call them “lat, long” , “lat, lon,” or “latitude and longitude,” and so on, same problem with other CRITICAL metadata.

What is the consequence of this? For instance, Brockmann Consult, partners of this [EOPF](https://cpm.pages.eopf.copernicus.eu/eopf-cpm/main/PSFDjan2024.html#eopf-storage-format), recently contributed to the generation of a very large ~2TB dataset in ZARR+ZIP format (you can check it [here](https://opara.zih.tu-dresden.de/items/bb45480f-f7d3-420f-85b8-b4993715b761/full)). When you download the [demo](https://opara.zih.tu-dresden.de/bitstreams/1221405a-ab9d-4dd8-b65e-b1b58d8f1423/download) (in Zarr+Zip format), decompress it, and load it into QGIS, this happens.

![ezgif-4-459e4813d1](https://github.com/user-attachments/assets/cc232ecf-003c-4f73-bcb1-a9e501e2d15c)

I’m downloading this dataset, and the patches for Oceania are showing up in Canada. Needless to say, ZARR+ZIP causes an error when you directly drag it into QGIS.

I think this decision only considers a very small sector of the EO community. **We are in a bubble, we are a minority**. What about the part of the community that doesn’t use Python or those who don’t code and rely on GIS software with less support, like ILWIS, IDRISI, or SPRING? Many communities in the Amazon in Peru and Brazil use SPRING/TerraLib, and all their raster data is in GeoTIFF v1.0. How are we going to teach them to use ZARR+Zipped now?

Have you considered how easy is to do a [zip bomb](https://en.wikipedia.org/wiki/Zip_bomb)?



