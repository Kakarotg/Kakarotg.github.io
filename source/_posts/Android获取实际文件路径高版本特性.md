---
title: Android获取实际文件路径高版本特性
---

## Android获取实际文件路径高版本特性

1.通过文件管理器获取系统文件的实际文件路径时，Android10以上版本，需先copy到自己apk目录下，否则获取不到，为空

### API 19-29  获取方法为
```java
if (isKitKat && DocumentsContract.isDocumentUri(context, uri)) {
    // ExternalStorageProvider
    if (isExternalStorageDocument(uri)) {
        final String docId = DocumentsContract.getDocumentId(uri);
        final String[] split = docId.split(":");
        final String type = split[0];

        if ("primary".equalsIgnoreCase(type)) {
            return Environment.getExternalStorageDirectory() + "/" + split[1];
        }

        // TODO handle non-primary volumes
    }
    // DownloadsProvider
    else if (isDownloadsDocument(uri)) {
        final String id = DocumentsContract.getDocumentId(uri);
        final Uri contentUri = ContentUris.withAppendedId(
                Uri.parse("content://downloads/public_downloads"), Long.valueOf(id));

        return getDataColumn(context, contentUri, null, null);
    }
    // MediaProvider
    else if (isMediaDocument(uri)) {
        final String docId = DocumentsContract.getDocumentId(uri);
        final String[] split = docId.split(":");
        final String type = split[0];
        Uri contentUri = null;
        if ("image".equals(type)) {
            contentUri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;
        } else if ("video".equals(type)) {
            contentUri = MediaStore.Video.Media.EXTERNAL_CONTENT_URI;
        } else if ("audio".equals(type)) {
            contentUri = MediaStore.Audio.Media.EXTERNAL_CONTENT_URI;
        }

        final String selection = "_id=?";
        final String[] selectionArgs = new String[] {
                split[1]
        };
        return getDataColumn(context, contentUri, selection, selectionArgs);
    }
    
public static String getDataColumn(Context context, Uri uri, String selection,
                                    String[] selectionArgs) {
    Cursor cursor = null;
    final String column = "_data";
    final String[] projection = {
            column
    };

    try {
        Log.d(TAG,"Query information : " + uri + Arrays.toString(projection) + selection + Arrays.toString(selectionArgs));
        cursor = context.getContentResolver().query(uri, projection, selection, selectionArgs,
                null);
        if (cursor != null && cursor.moveToFirst()) {
            Log.d(TAG,"Find File Sucessfully");
            final int index = cursor.getColumnIndexOrThrow(column);
            return cursor.getString(index);
        }
    } finally {
        if (cursor != null)
            cursor.close();
    }
    return null;
}
```

### API 29以上获取方法为 
```java
private static String uriToFileApiQ(Context context, Uri uri) {
    File file = null;
    //android10以上转换
    if (uri.getScheme().equals(ContentResolver.SCHEME_FILE)) {
        file = new File(uri.getPath());
    } else if (uri.getScheme().equals(ContentResolver.SCHEME_CONTENT)) {
        //把文件复制到沙盒目录
        ContentResolver contentResolver = context.getContentResolver();
        Cursor cursor = contentResolver.query(uri, null, null, null, null);
        if (cursor.moveToFirst()) {
            @SuppressLint("Range") String displayName = cursor.getString(cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME));
            try {
                InputStream is = contentResolver.openInputStream(uri);
                File cache = new File(context.getExternalCacheDir().getAbsolutePath(), Math.round((Math.random() + 1) * 1000) + displayName);
                FileOutputStream fos = new FileOutputStream(cache);
                FileUtils.copy(is, fos);
                file = cache;

                fos.close();
                is.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    if (file != null) {
        Log.d(TAG,"Temp File is located in " + file.getAbsolutePath());
        return file.getAbsolutePath();
    } else {
        return null;
    }
}
```
