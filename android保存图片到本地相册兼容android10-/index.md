# Android保存图片到本地相册（兼容Android10+）

## 背景
有收到用户反馈，存在图片保存到相册Crash以及相册中找不到到问题，看了下是因为目前App还未适配Android10以上的设备，所以一些Api在一些厂商Android10以上的设备会存在一些问题。

为了能够使用到新的Api来实现保存到相册的逻辑，为了迅速修复保存相册的问题我们可以先升级compileApi到30，以至于我们代码中可以正常访问到新的Api，而targetApi可以先不升，如果升了不做充分的兼容性测试肯定是会存在大大小小的一些问题的。

## 代码实现
具体的实现可以看以下代码，看目前网上的代码实现可能会存在一些缺陷，这份代码主要是参考Google官方文档以及结合网上的一些文章，并且通过了一些主流设备的兼容性测试的。

```
import android.content.ContentResolver
import android.content.ContentUris
import android.content.ContentValues
import android.content.Context
import android.content.Intent
import android.graphics.Bitmap
import android.net.Uri
import android.os.Build
import android.os.Environment
import android.provider.MediaStore
import com.tencent.videocut.utils.log.Logger
import java.io.File
import java.io.FileNotFoundException
import java.io.OutputStream
import java.util.Locale
import java.util.concurrent.TimeUnit

private const val TAG = "ImageExt"
private const val SUFFIX_JPEG = ".jpeg"
private const val SUFFIX_JPG = ".jpg"
private const val SUFFIX_PNG = ".png"
private const val SUFFIX_WEBP = ".webp"
private const val MIME_TYPE_JPEG = "image/jpeg"
private const val MIME_TYPE_PNG = "image/png"
private const val MIME_TYPE_WEBP = "image/webp"

private data class InsertResult(
    val uri: Uri, // 插入的Uri
    val file: File?,// 插入的文件，Android Q以下需要用于获取文件名
)

/**
 * 保存Bitmap到相册的Pictures文件夹
 *
 * 参考：https://developer.android.google.cn/training/data-storage/shared/media
 *
 * @param context 上下文
 * @param fileName 文件名，需要携带后缀
 * @param quality 图片压缩质量
 * @return 本地图片的Uri，返回null则代表保存失败，否则成功
 */
suspend fun Bitmap.saveToAlbum(
    context: Context,
    fileName: String,
    quality: Int = 75,
): Uri? {
    // 插入图片信息
    val insertResult = context.contentResolver.insertMediaImage(fileName)
    if (insertResult == null) {
        Logger.e(TAG, "[saveToAlbum] insert media image fail")
        return null
    }
    // 保存图片并压缩
    val outputStream = insertResult.uri.outputStream(context.contentResolver)
    if (outputStream == null) {
        Logger.e(TAG, "[saveToAlbum] outputStream is null")
        return null
    }
    outputStream.use { compress(fileName.getBitmapFormat(), quality, it) }
    // 同步到相册
    insertResult.syncToAlbum(context)
    return insertResult.uri
}

private fun String.getBitmapFormat(): Bitmap.CompressFormat {
    val fileName = toLowerCase(Locale.getDefault())
    return when {
        fileName.endsWith(SUFFIX_PNG) -> Bitmap.CompressFormat.PNG
        fileName.endsWith(SUFFIX_JPG) || fileName.endsWith(SUFFIX_JPEG) -> Bitmap.CompressFormat.JPEG
        fileName.endsWith(SUFFIX_WEBP) -> Bitmap.CompressFormat.WEBP
        else -> Bitmap.CompressFormat.PNG
    }
}

private fun Uri.outputStream(resolver: ContentResolver): OutputStream? {
    return try {
        resolver.openOutputStream(this)
    } catch (e: FileNotFoundException) {
        Logger.e(TAG, "[outputStream]", e)
        null
    }
}

@Suppress("DEPRECATION")
private fun InsertResult.syncToAlbum(context: Context) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
        val imageValues = ContentValues()
        // 写入文件大小
        file?.let { imageValues.put(MediaStore.Images.Media.SIZE, it.length()) }
        context.contentResolver.update(uri, imageValues, null, null)
        // 通知媒体库更新
        context.sendBroadcast(Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE, uri))
    } else {
        val imageValues = ContentValues()
        // Android Q添加了IS_PENDING状态，为0时其他应用才可见
        imageValues.put(MediaStore.Images.Media.IS_PENDING, 0)
        context.contentResolver.update(uri, imageValues, null, null)
    }
}

private fun String.getMimeType(): String? {
    val fileName = toLowerCase(Locale.getDefault())
    return when {
        fileName.endsWith(SUFFIX_PNG) -> MIME_TYPE_PNG
        fileName.endsWith(SUFFIX_JPG) || fileName.endsWith(SUFFIX_JPEG) -> MIME_TYPE_JPEG
        fileName.endsWith(SUFFIX_WEBP) -> MIME_TYPE_WEBP
        else -> null
    }
}

/**
 * 插入图片到媒体库
 */
@Suppress("DEPRECATION")
private fun ContentResolver.insertMediaImage(fileName: String): InsertResult? {
    val imageValues = ContentValues().apply {
        fileName.getMimeType()?.let {
            put(MediaStore.Images.Media.MIME_TYPE, it)
        }
        TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis()).let {
            put(MediaStore.Images.Media.DATE_ADDED, it)
            put(MediaStore.Images.Media.DATE_MODIFIED, it)
        }
    }
    val collection: Uri
    var file: File? = null
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        imageValues.apply {
            put(MediaStore.Images.Media.DISPLAY_NAME, fileName)
            put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }
        // 高版本直接插入，会自动重命名
        collection = MediaStore.Images.Media.getContentUri(MediaStore.VOLUME_EXTERNAL_PRIMARY)
    } else {
        val pictureDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)
        if (!pictureDir.exists() && !pictureDir.mkdirs()) {
            Logger.e(TAG, "[insertMediaImage] can't create Pictures directory")
            return null
        }
        // 文件路径查重，重复的话在文件名后拼接数字
        var imageFile = File(pictureDir, fileName)
        var queryUri = queryMediaImageAboveQ(imageFile.absolutePath)
        var suffix = 1
        while (queryUri != null) {
            val newName = imageFile.nameWithoutExtension + "(${suffix++})." + imageFile.extension
            imageFile = File(pictureDir, newName)
            queryUri = queryMediaImageAboveQ(imageFile.absolutePath)
        }
        imageValues.apply {
            put(MediaStore.Images.Media.DISPLAY_NAME, imageFile.name)
            put(MediaStore.Images.Media.DATA, imageFile.absolutePath)
        }
        file = imageFile
        collection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
    }
    val insertUrl = insert(collection, imageValues)
    if (insertUrl == null) {
        Logger.e(TAG, "[insertMediaImage] insert error.collection = $collection, imageValues = $imageValues")
        return null
    }
    return InsertResult(insertUrl, file)
}

/**
 * Android Q以下版本，查询媒体库中当前路径是否存在
 * @return Uri 返回null时说明不存在，可以进行图片插入逻辑
 */
@Suppress("DEPRECATION")
private fun ContentResolver.queryMediaImageAboveQ(imagePath: String): Uri? {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) return null
    val imageFile = File(imagePath)
    if (imageFile.canRead() && imageFile.exists()) {
        Logger.i(TAG, "[queryMediaImageAboveQ] path: $imagePath exists")
        return Uri.fromFile(imageFile)
    }
    val collection = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
    query(
        collection,
        arrayOf(MediaStore.Images.Media._ID, MediaStore.Images.Media.DATA),
        "${MediaStore.Images.Media.DATA} == ?",
        arrayOf(imagePath),
        null
    )?.use { cursor ->
        while (cursor.moveToNext()) {
            val idColumn = cursor.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
            val id = cursor.getLong(idColumn)
            return ContentUris.withAppendedId(collection, id).also {
                Logger.i(TAG, "[queryMediaImageAboveQ] $imagePath exists uri: $it")
            }
        }
    }
    return null
}
```

## 参考
[Access media files from shared storage](https://developer.android.google.cn/training/data-storage/shared/media)
