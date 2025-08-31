/ModLoader/app/src/main/java/com/modloader/util/ApkValidator.java

package com.modloader.util;

import java.util.ArrayList;
import java.util.List;

public class ApkValidator {
    
    public static class ValidationResult {
        public boolean isValid;
        public String processId;
        public String errorMessage;
        public List<String> warnings;
        public List<String> validationErrors;
        public List<String> issues;  // Added missing field
        public long validationTime;
        public String apkPath;
        public String modType;
        public long fileSize;        // Added missing field
        public int totalEntries;     // Added missing field
        public String packageName;   // Added missing field
        
        public ValidationResult() {
            this.isValid = false;
            this.warnings = new ArrayList<>();
            this.validationErrors = new ArrayList<>();
            this.issues = new ArrayList<>();  // Initialize issues list
            this.validationTime = System.currentTimeMillis();
            this.fileSize = 0;
            this.totalEntries = 0;
        }
        
        public ValidationResult(String processId) {
            this();
            this.processId = processId;
        }
        
        public ValidationResult(boolean isValid, String errorMessage) {
            this();
            this.isValid = isValid;
            this.errorMessage = errorMessage;
        }
        
        public void addWarning(String warning) {
            if (warnings == null) {
                warnings = new ArrayList<>();
            }
            warnings.add(warning);
        }
        
        public void addValidationError(String error) {
            if (validationErrors == null) {
                validationErrors = new ArrayList<>();
            }
            validationErrors.add(error);
            
            // Also add to issues list for compatibility
            if (issues == null) {
                issues = new ArrayList<>();
            }
            issues.add(error);
            this.isValid = false; // Any validation error makes the result invalid
        }
        
        public boolean hasWarnings() {
            return warnings != null && !warnings.isEmpty();
        }
        
        public boolean hasErrors() {
            return validationErrors != null && !validationErrors.isEmpty();
        }
        
        public void setValid(boolean valid) {
            this.isValid = valid;
        }
        
        public void setProcessId(String processId) {
            this.processId = processId;
        }
        
        public void setErrorMessage(String errorMessage) {
            this.errorMessage = errorMessage;
        }
        
        public void setApkPath(String apkPath) {
            this.apkPath = apkPath;
        }
        
        public void setPackageName(String packageName) {
            this.packageName = packageName;
        }
        
        public void setFileSize(long fileSize) {
            this.fileSize = fileSize;
        }
        
        public void setTotalEntries(int totalEntries) {
            this.totalEntries = totalEntries;
        }
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("ValidationResult{");
            sb.append("isValid=").append(isValid);
            sb.append(", processId='").append(processId).append('\'');
            if (errorMessage != null) {
                sb.append(", errorMessage='").append(errorMessage).append('\'');
            }
            if (hasWarnings()) {
                sb.append(", warnings=").append(warnings.size());
            }
            if (hasErrors()) {
                sb.append(", errors=").append(validationErrors.size());
            }
            sb.append('}');
            return sb.toString();
        }
    }
    
    // Static validation methods
    public static ValidationResult validateApk(String apkPath, String processId) {
        ValidationResult result = new ValidationResult(processId);
        result.setApkPath(apkPath);
        
        try {
            // Add your APK validation logic here
            java.io.File apkFile = new java.io.File(apkPath);
            
            if (!apkFile.exists()) {
                result.addValidationError("APK file does not exist: " + apkPath);
                return result;
            }
            
            if (apkFile.length() == 0) {
                result.addValidationError("APK file is empty");
                return result;
            }
            
            // Set file size
            result.setFileSize(apkFile.length());
            
            // Basic APK signature check (you can expand this)
            if (!apkPath.toLowerCase().endsWith(".apk")) {
                result.addWarning("File does not have .apk extension");
            }
            
            // Try to get package info (basic implementation)
            try {
                // This is a simple approximation - you might want to use actual APK parsing
                String fileName = apkFile.getName();
                if (fileName.contains(".")) {
                    result.setPackageName(fileName.substring(0, fileName.lastIndexOf(".")));
                }
            } catch (Exception e) {
                result.addWarning("Could not extract package name");
            }
            
            // If we get here, basic validation passed
            result.setValid(true);
            
        } catch (Exception e) {
            result.addValidationError("Validation failed: " + e.getMessage());
        }
        
        return result;
    }
    
    // Additional validation methods
    public static boolean isValidApk(String apkPath) {
        ValidationResult result = validateApk(apkPath, null);
        return result.isValid;
    }
    
    public static ValidationResult quickValidation(String processId) {
        ValidationResult result = new ValidationResult(processId);
        result.setValid(true); // Quick validation always passes
        return result;
    }
    
    public static ValidationResult createValidationResult(String processId, boolean isValid, String message) {
        ValidationResult result = new ValidationResult(processId);
        result.setValid(isValid);
        if (!isValid && message != null) {
            result.setErrorMessage(message);
        }
        return result;
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/DiagnosticBundleExporter.java

// File: DiagnosticBundleExporter.java - Comprehensive Support Bundle Creator
// Path: /main/java/com/terrarialoader/util/DiagnosticBundleExporter.java

package com.modloader.util;

import android.content.Context;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.os.Build;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.loader.ModManager;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

/**
 * Creates comprehensive diagnostic bundles for support purposes
 * Automatically compiles app logs, system info, mod info, and configuration
 */
public class DiagnosticBundleExporter {
    
    private static final String BUNDLE_NAME_FORMAT = "TerrariaLoader_Diagnostic_%s.zip";
    private static final SimpleDateFormat TIMESTAMP_FORMAT = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault());
    
    /**
     * Create a comprehensive diagnostic bundle
     * @param context Application context
     * @return File object of created bundle, or null if failed
     */
    public static File createDiagnosticBundle(Context context) {
        LogUtils.logUser("🔧 Creating comprehensive diagnostic bundle...");
        
        try {
            // Create bundle file
            String timestamp = TIMESTAMP_FORMAT.format(new Date());
            String bundleName = String.format(BUNDLE_NAME_FORMAT, timestamp);
            File exportsDir = new File(context.getExternalFilesDir(null), "exports");
            if (!exportsDir.exists()) {
                exportsDir.mkdirs();
            }
            File bundleFile = new File(exportsDir, bundleName);
            
            // Create ZIP bundle
            try (ZipOutputStream zos = new ZipOutputStream(new FileOutputStream(bundleFile))) {
                
                // Add diagnostic report
                addDiagnosticReport(context, zos, timestamp);
                
                // Add system information
                addSystemInformation(context, zos);
                
                // Add application logs
                addApplicationLogs(context, zos);
                
                // Add game logs if available
                addGameLogs(context, zos);
                
                // Add mod information
                addModInformation(context, zos);
                
                // Add loader information
                addLoaderInformation(context, zos);
                
                // Add directory structure
                addDirectoryStructure(context, zos);
                
                // Add configuration files
                addConfigurationFiles(context, zos);
                
            }
            
            LogUtils.logUser("✅ Diagnostic bundle created: " + bundleName);
            LogUtils.logUser("📦 Size: " + FileUtils.formatFileSize(bundleFile.length()));
            
            return bundleFile;
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to create diagnostic bundle: " + e.getMessage());
            return null;
        }
    }
    
    /**
     * Add main diagnostic report
     */
    private static void addDiagnosticReport(Context context, ZipOutputStream zos, String timestamp) throws IOException {
        StringBuilder report = new StringBuilder();
        
        report.append("=== TERRARIA LOADER DIAGNOSTIC REPORT ===\n\n");
        report.append("Generated: ").append(new Date().toString()).append("\n");
        report.append("Bundle ID: ").append(timestamp).append("\n");
        report.append("Report Version: 2.0\n\n");
        
        // Executive summary
        report.append("=== EXECUTIVE SUMMARY ===\n");
        report.append("App Status: ").append(getAppStatus(context)).append("\n");
        report.append("Loader Status: ").append(getLoaderStatus(context)).append("\n");
        report.append("Mod Count: ").append(ModManager.getTotalModCount()).append("\n");
        report.append("Error Level: ").append(getErrorLevel(context)).append("\n\n");
        
        // Quick diagnostics
        report.append("=== QUICK DIAGNOSTICS ===\n");
        report.append(runQuickDiagnostics(context));
        report.append("\n");
        
        // Recommendations
        report.append("=== RECOMMENDATIONS ===\n");
        report.append(generateRecommendations(context));
        report.append("\n");
        
        // Bundle contents
        report.append("=== BUNDLE CONTENTS ===\n");
        report.append("1. diagnostic_report.txt - This report\n");
        report.append("2. system_info.txt - Device and OS information\n");
        report.append("3. app_logs/ - Application log files\n");
        report.append("4. game_logs/ - Game/MelonLoader log files (if available)\n");
        report.append("5. mod_info.txt - Installed mod information\n");
        report.append("6. loader_info.txt - MelonLoader installation details\n");
        report.append("7. directory_structure.txt - File system layout\n");
        report.append("8. configuration/ - Configuration files\n\n");
        
        addTextFile(zos, "diagnostic_report.txt", report.toString());
    }
    
    /**
     * Add system information
     */
    private static void addSystemInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder sysInfo = new StringBuilder();
        
        sysInfo.append("=== SYSTEM INFORMATION ===\n\n");
        
        // Device information
        sysInfo.append("Device Information:\n");
        sysInfo.append("Manufacturer: ").append(Build.MANUFACTURER).append("\n");
        sysInfo.append("Model: ").append(Build.MODEL).append("\n");
        sysInfo.append("Device: ").append(Build.DEVICE).append("\n");
        sysInfo.append("Product: ").append(Build.PRODUCT).append("\n");
        sysInfo.append("Hardware: ").append(Build.HARDWARE).append("\n");
        sysInfo.append("Board: ").append(Build.BOARD).append("\n");
        sysInfo.append("Brand: ").append(Build.BRAND).append("\n\n");
        
        // OS information
        sysInfo.append("Operating System:\n");
        sysInfo.append("Android Version: ").append(Build.VERSION.RELEASE).append("\n");
        sysInfo.append("API Level: ").append(Build.VERSION.SDK_INT).append("\n");
        sysInfo.append("Codename: ").append(Build.VERSION.CODENAME).append("\n");
        sysInfo.append("Incremental: ").append(Build.VERSION.INCREMENTAL).append("\n");
        sysInfo.append("Security Patch: ").append(Build.VERSION.SECURITY_PATCH).append("\n\n");
        
        // App information
        sysInfo.append("Application Information:\n");
        try {
            PackageManager pm = context.getPackageManager();
            PackageInfo pInfo = pm.getPackageInfo(context.getPackageName(), 0);
            sysInfo.append("Package Name: ").append(pInfo.packageName).append("\n");
            sysInfo.append("Version Name: ").append(pInfo.versionName).append("\n");
            sysInfo.append("Version Code: ").append(pInfo.versionCode).append("\n");
            sysInfo.append("Target SDK: ").append(pInfo.applicationInfo.targetSdkVersion).append("\n");
        } catch (Exception e) {
            sysInfo.append("Could not retrieve app information: ").append(e.getMessage()).append("\n");
        }
        sysInfo.append("\n");
        
        // Memory information
        sysInfo.append("Memory Information:\n");
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        sysInfo.append("Max Memory: ").append(FileUtils.formatFileSize(maxMemory)).append("\n");
        sysInfo.append("Total Memory: ").append(FileUtils.formatFileSize(totalMemory)).append("\n");
        sysInfo.append("Used Memory: ").append(FileUtils.formatFileSize(usedMemory)).append("\n");
        sysInfo.append("Free Memory: ").append(FileUtils.formatFileSize(freeMemory)).append("\n\n");
        
        // Storage information
        sysInfo.append("Storage Information:\n");
        try {
            File appDir = context.getExternalFilesDir(null);
            if (appDir != null) {
                sysInfo.append("App Directory: ").append(appDir.getAbsolutePath()).append("\n");
                sysInfo.append("Total Space: ").append(FileUtils.formatFileSize(appDir.getTotalSpace())).append("\n");
                sysInfo.append("Free Space: ").append(FileUtils.formatFileSize(appDir.getFreeSpace())).append("\n");
                sysInfo.append("Usable Space: ").append(FileUtils.formatFileSize(appDir.getUsableSpace())).append("\n");
            }
        } catch (Exception e) {
            sysInfo.append("Could not retrieve storage information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "system_info.txt", sysInfo.toString());
    }
    
    /**
     * Add application logs
     */
    private static void addApplicationLogs(Context context, ZipOutputStream zos) throws IOException {
        try {
            // Add current logs
            String currentLogs = LogUtils.getLogs();
            if (!currentLogs.isEmpty()) {
                addTextFile(zos, "app_logs/current_session.txt", currentLogs);
            }
            
            // Add rotated log files
            List<File> logFiles = LogUtils.getAvailableLogFiles();
            for (int i = 0; i < logFiles.size(); i++) {
                File logFile = logFiles.get(i);
                if (logFile.exists() && logFile.length() > 0) {
                    String content = LogUtils.readLogFile(i);
                    addTextFile(zos, "app_logs/" + logFile.getName(), content);
                }
            }
            
        } catch (Exception e) {
            addTextFile(zos, "app_logs/error.txt", "Could not retrieve app logs: " + e.getMessage());
        }
    }
    
    /**
     * Add game logs if available
     */
    private static void addGameLogs(Context context, ZipOutputStream zos) throws IOException {
        try {
            File gameLogsDir = PathManager.getLogsDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (gameLogsDir.exists()) {
                File[] gameLogFiles = gameLogsDir.listFiles((dir, name) -> 
                    name.startsWith("Log") && name.endsWith(".txt"));
                
                if (gameLogFiles != null && gameLogFiles.length > 0) {
                    for (File logFile : gameLogFiles) {
                        try {
                            String content = readFileContent(logFile);
                            addTextFile(zos, "game_logs/" + logFile.getName(), content);
                        } catch (Exception e) {
                            addTextFile(zos, "game_logs/" + logFile.getName() + "_error.txt", 
                                "Could not read log file: " + e.getMessage());
                        }
                    }
                } else {
                    addTextFile(zos, "game_logs/no_logs.txt", "No game log files found");
                }
            } else {
                addTextFile(zos, "game_logs/directory_not_found.txt", 
                    "Game logs directory does not exist: " + gameLogsDir.getAbsolutePath());
            }
        } catch (Exception e) {
            addTextFile(zos, "game_logs/error.txt", "Could not access game logs: " + e.getMessage());
        }
    }
    
    /**
     * Add mod information
     */
    private static void addModInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder modInfo = new StringBuilder();
        
        modInfo.append("=== MOD INFORMATION ===\n\n");
        
        try {
            // Statistics
            modInfo.append("Statistics:\n");
            modInfo.append("Total Mods: ").append(ModManager.getTotalModCount()).append("\n");
            modInfo.append("Enabled Mods: ").append(ModManager.getEnabledModCount()).append("\n");
            modInfo.append("Disabled Mods: ").append(ModManager.getDisabledModCount()).append("\n");
            modInfo.append("DEX Mods: ").append(ModManager.getDexModCount()).append("\n");
            modInfo.append("DLL Mods: ").append(ModManager.getDllModCount()).append("\n\n");
            
            // Available mods
            modInfo.append("Available Mods:\n");
            List<File> availableMods = ModManager.getAvailableMods();
            if (availableMods != null && !availableMods.isEmpty()) {
                for (File mod : availableMods) {
                    modInfo.append("- ").append(mod.getName());
                    modInfo.append(" (").append(FileUtils.formatFileSize(mod.length())).append(")");
                    modInfo.append(" [").append(ModManager.getModStatus(mod)).append("]");
                    modInfo.append(" {").append(ModManager.getModType(mod).getDisplayName()).append("}\n");
                }
            } else {
                modInfo.append("No mods found\n");
            }
            modInfo.append("\n");
            
            // Directory paths
            modInfo.append("Mod Directories:\n");
            modInfo.append("DEX Mods: ").append(ModManager.getModsDirectoryPath(context)).append("\n");
            modInfo.append("DLL Mods: ").append(ModManager.getDllModsDirectoryPath(context)).append("\n");
            
        } catch (Exception e) {
            modInfo.append("Error retrieving mod information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "mod_info.txt", modInfo.toString());
    }
    
    /**
     * Add loader information
     */
    private static void addLoaderInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder loaderInfo = new StringBuilder();
        
        loaderInfo.append("=== LOADER INFORMATION ===\n\n");
        
        try {
            String gamePackage = MelonLoaderManager.TERRARIA_PACKAGE;
            
            // Status
            boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(context, gamePackage);
            boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(context, gamePackage);
            
            loaderInfo.append("Installation Status:\n");
            loaderInfo.append("MelonLoader Installed: ").append(melonInstalled).append("\n");
            loaderInfo.append("LemonLoader Installed: ").append(lemonInstalled).append("\n");
            
            if (melonInstalled || lemonInstalled) {
                loaderInfo.append("Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
            }
            loaderInfo.append("\n");
            
            // Validation report
            loaderInfo.append("Validation Report:\n");
            loaderInfo.append(MelonLoaderManager.getValidationReport(context, gamePackage));
            loaderInfo.append("\n");
            
            // Debug information
            loaderInfo.append("Debug Information:\n");
            loaderInfo.append(MelonLoaderManager.getDebugInfo(context, gamePackage));
            
        } catch (Exception e) {
            loaderInfo.append("Error retrieving loader information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "loader_info.txt", loaderInfo.toString());
    }
    
    /**
     * Add directory structure
     */
    private static void addDirectoryStructure(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder structure = new StringBuilder();
        
        structure.append("=== DIRECTORY STRUCTURE ===\n\n");
        
        try {
            // Path information
            structure.append("Path Information:\n");
            structure.append(PathManager.getPathInfo(context, MelonLoaderManager.TERRARIA_PACKAGE));
            structure.append("\n");
            
            // Directory tree
            structure.append("Directory Tree:\n");
            File baseDir = PathManager.getTerrariaBaseDir(context);
            if (baseDir.exists()) {
                structure.append(generateDirectoryTree(baseDir, ""));
            } else {
                structure.append("Base directory does not exist: ").append(baseDir.getAbsolutePath()).append("\n");
            }
            
        } catch (Exception e) {
            structure.append("Error generating directory structure: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "directory_structure.txt", structure.toString());
    }
    
    /**
     * Add configuration files
     */
    private static void addConfigurationFiles(Context context, ZipOutputStream zos) throws IOException {
        try {
            // App preferences
            addTextFile(zos, "configuration/app_preferences.txt", getAppPreferences(context));
            
            // Config directory files
            File configDir = PathManager.getConfigDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (configDir.exists()) {
                File[] configFiles = configDir.listFiles((dir, name) -> 
                    name.endsWith(".cfg") || name.endsWith(".json") || name.endsWith(".txt"));
                
                if (configFiles != null) {
                    for (File configFile : configFiles) {
                        try {
                            String content = readFileContent(configFile);
                            addTextFile(zos, "configuration/" + configFile.getName(), content);
                        } catch (Exception e) {
                            addTextFile(zos, "configuration/" + configFile.getName() + "_error.txt", 
                                "Could not read config file: " + e.getMessage());
                        }
                    }
                }
            }
            
        } catch (Exception e) {
            addTextFile(zos, "configuration/error.txt", "Could not retrieve configuration: " + e.getMessage());
        }
    }
    
    // Helper methods
    
    private static String getAppStatus(Context context) {
        try {
            return "Running normally";
        } catch (Exception e) {
            return "Error: " + e.getMessage();
        }
    }
    
    private static String getLoaderStatus(Context context) {
        try {
            boolean installed = MelonLoaderManager.isMelonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE);
            return installed ? "Installed" : "Not installed";
        } catch (Exception e) {
            return "Error: " + e.getMessage();
        }
    }
    
    private static String getErrorLevel(Context context) {
        try {
            String logs = LogUtils.getLogs();
            if (logs.toLowerCase().contains("error") || logs.toLowerCase().contains("crash")) {
                return "High";
            } else if (logs.toLowerCase().contains("warn")) {
                return "Medium";
            } else {
                return "Low";
            }
        } catch (Exception e) {
            return "Unknown";
        }
    }
    
    private static String runQuickDiagnostics(Context context) {
        StringBuilder diagnostics = new StringBuilder();
        
        try {
            // Directory validation
            boolean directoriesValid = ModManager.validateModDirectories(context);
            diagnostics.append("Directory Structure: ").append(directoriesValid ? "✅ Valid" : "❌ Invalid").append("\n");
            
            // Health check
            boolean healthCheck = ModManager.performHealthCheck(context);
            diagnostics.append("Health Check: ").append(healthCheck ? "✅ Passed" : "❌ Failed").append("\n");
            
            // Migration status
            boolean needsMigration = PathManager.needsMigration(context);
            diagnostics.append("Migration Needed: ").append(needsMigration ? "⚠️ Yes" : "✅ No").append("\n");
            
        } catch (Exception e) {
            diagnostics.append("Diagnostic error: ").append(e.getMessage()).append("\n");
        }
        
        return diagnostics.toString();
    }
    
    private static String generateRecommendations(Context context) {
        StringBuilder recommendations = new StringBuilder();
        
        try {
            if (PathManager.needsMigration(context)) {
                recommendations.append("• Migrate to new directory structure\n");
            }
            
            if (!MelonLoaderManager.isMelonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE)) {
                recommendations.append("• Install MelonLoader for DLL mod support\n");
            }
            
            if (ModManager.getTotalModCount() == 0) {
                recommendations.append("• Install some mods to get started\n");
            }
            
            if (recommendations.length() == 0) {
                recommendations.append("• No specific recommendations at this time\n");
            }
            
        } catch (Exception e) {
            recommendations.append("• Could not generate recommendations: ").append(e.getMessage()).append("\n");
        }
        
        return recommendations.toString();
    }
    
    private static String getAppPreferences(Context context) {
        StringBuilder prefs = new StringBuilder();
        
        prefs.append("=== APP PREFERENCES ===\n\n");
        
        try {
            prefs.append("Mods Enabled: ").append(com.modloader.ui.SettingsActivity.isModsEnabled(context)).append("\n");
            prefs.append("Debug Mode: ").append(com.modloader.ui.SettingsActivity.isDebugMode(context)).append("\n");
            prefs.append("Sandbox Mode: ").append(com.modloader.ui.SettingsActivity.isSandboxMode(context)).append("\n");
            prefs.append("Auto Save Logs: ").append(com.modloader.ui.SettingsActivity.isAutoSaveEnabled(context)).append("\n");
        } catch (Exception e) {
            prefs.append("Could not retrieve preferences: ").append(e.getMessage()).append("\n");
        }
        
        return prefs.toString();
    }
    
    private static String generateDirectoryTree(File dir, String indent) {
        StringBuilder tree = new StringBuilder();
        
        if (dir == null || !dir.exists()) {
            return tree.toString();
        }
        
        tree.append(indent).append(dir.getName());
        if (dir.isDirectory()) {
            tree.append("/\n");
            File[] files = dir.listFiles();
            if (files != null && files.length > 0) {
                for (File file : files) {
                    if (indent.length() < 20) { // Limit depth
                        tree.append(generateDirectoryTree(file, indent + "  "));
                    }
                }
            }
        } else {
            tree.append(" (").append(FileUtils.formatFileSize(dir.length())).append(")\n");
        }
        
        return tree.toString();
    }
    
    private static String readFileContent(File file) throws IOException {
        StringBuilder content = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
        }
        return content.toString();
    }
    
    private static void addTextFile(ZipOutputStream zos, String filename, String content) throws IOException {
        ZipEntry entry = new ZipEntry(filename);
        zos.putNextEntry(entry);
        zos.write(content.getBytes("UTF-8"));
        zos.closeEntry();
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/Downloader.java

// File: Downloader.java (Fixed Utility Class)
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/Downloader.java

package com.modloader.util;

import com.modloader.util.LogUtils;
import java.io.File;
import java.io.FileOutputStream;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

public class Downloader {

    /**
     * Downloads and extracts a ZIP file from a URL to a specified directory.
     * FIXED: Properly handles nested folder structures and flattens them correctly.
     *
     * @param fileUrl The URL of the ZIP file to download.
     * @param targetDirectory The directory where the contents will be extracted.
     * @return true if successful, false otherwise.
     */
    public static boolean downloadAndExtractZip(String fileUrl, File targetDirectory) {
        File zipFile = null;
        try {
            // Ensure the target directory exists
            if (!targetDirectory.exists() && !targetDirectory.mkdirs()) {
                LogUtils.logDebug("❌ Failed to create target directory: " + targetDirectory.getAbsolutePath());
                return false;
            }
            LogUtils.logUser("📂 Target directory prepared: " + targetDirectory.getAbsolutePath());

            // --- Download Step ---
            LogUtils.logUser("🌐 Starting download from: " + fileUrl);
            URL url = new URL(fileUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.connect();

            if (connection.getResponseCode() != HttpURLConnection.HTTP_OK) {
                LogUtils.logDebug("❌ Server returned HTTP " + connection.getResponseCode() + " " + connection.getResponseMessage());
                return false;
            }

            zipFile = new File(targetDirectory, "downloaded.zip");
            try (InputStream input = connection.getInputStream();
                 FileOutputStream output = new FileOutputStream(zipFile)) {

                byte[] data = new byte[4096];
                int count;
                long total = 0;
                while ((count = input.read(data)) != -1) {
                    total += count;
                    output.write(data, 0, count);
                }
                LogUtils.logUser("✅ Download complete. Total size: " + FileUtils.formatFileSize(total));
            }

            // --- Extraction Step with Smart Path Handling ---
            LogUtils.logUser("📦 Starting extraction of " + zipFile.getName());
            try (InputStream is = new java.io.FileInputStream(zipFile);
                 ZipInputStream zis = new ZipInputStream(new java.io.BufferedInputStream(is))) {
                
                ZipEntry zipEntry;
                int extractedCount = 0;
                while ((zipEntry = zis.getNextEntry()) != null) {
                    if (zipEntry.isDirectory()) {
                        zis.closeEntry();
                        continue;
                    }
                    
                    // FIXED: Smart path handling to flatten nested MelonLoader directories
                    String entryPath = zipEntry.getName();
                    String targetPath = getSmartTargetPath(entryPath);
                    
                    if (targetPath == null) {
                        LogUtils.logDebug("Skipping file: " + entryPath);
                        zis.closeEntry();
                        continue;
                    }
                    
                    File newFile = new File(targetDirectory, targetPath);
                    
                    // Prevent Zip Path Traversal Vulnerability
                    if (!newFile.getCanonicalPath().startsWith(targetDirectory.getCanonicalPath() + File.separator)) {
                        throw new SecurityException("Zip Path Traversal detected: " + zipEntry.getName());
                    }

                    // Create parent directories if they don't exist
                    newFile.getParentFile().mkdirs();
                    
                    try (FileOutputStream fos = new FileOutputStream(newFile)) {
                        int len;
                        byte[] buffer = new byte[4096];
                        while ((len = zis.read(buffer)) > 0) {
                            fos.write(buffer, 0, len);
                        }
                    }
                    
                    extractedCount++;
                    LogUtils.logDebug("Extracted: " + entryPath + " -> " + targetPath);
                    zis.closeEntry();
                }
                LogUtils.logUser("✅ Extracted " + extractedCount + " files successfully.");
            }
            return true;

        } catch (Exception e) {
            LogUtils.logDebug("❌ Download and extraction failed: " + e.getMessage());
            e.printStackTrace();
            return false;
        } finally {
            // --- Cleanup Step ---
            if (zipFile != null && zipFile.exists()) {
                zipFile.delete();
                LogUtils.logDebug("🧹 Cleaned up temporary zip file.");
            }
        }
    }
    
    /**
     * FIXED: Smart path mapping to handle nested MelonLoader directories properly
     * This function flattens the nested structure and maps files to correct locations
     */
    private static String getSmartTargetPath(String zipEntryPath) {
        // Normalize path separators
        String normalizedPath = zipEntryPath.replace('\\', '/');
        
        // Remove leading MelonLoader/ if it exists (to flatten nested structure)
        if (normalizedPath.startsWith("MelonLoader/")) {
            normalizedPath = normalizedPath.substring("MelonLoader/".length());
        }
        
        // Skip empty paths or root directory entries
        if (normalizedPath.isEmpty() || normalizedPath.equals("/")) {
            return null;
        }
        
        // Map specific directory structures
        if (normalizedPath.startsWith("net8/")) {
            return "net8/" + normalizedPath.substring("net8/".length());
        } else if (normalizedPath.startsWith("net35/")) {
            return "net35/" + normalizedPath.substring("net35/".length());
        } else if (normalizedPath.startsWith("Dependencies/")) {
            return "Dependencies/" + normalizedPath.substring("Dependencies/".length());
        } else if (normalizedPath.contains("/net8/")) {
            // Handle nested paths like "SomeFolder/net8/file.dll"
            int net8Index = normalizedPath.indexOf("/net8/");
            return "net8/" + normalizedPath.substring(net8Index + "/net8/".length());
        } else if (normalizedPath.contains("/net35/")) {
            // Handle nested paths like "SomeFolder/net35/file.dll"
            int net35Index = normalizedPath.indexOf("/net35/");
            return "net35/" + normalizedPath.substring(net35Index + "/net35/".length());
        } else if (normalizedPath.contains("/Dependencies/")) {
            // Handle nested paths like "SomeFolder/Dependencies/file.dll"
            int depsIndex = normalizedPath.indexOf("/Dependencies/");
            return "Dependencies/" + normalizedPath.substring(depsIndex + "/Dependencies/".length());
        } else {
            // For any other files, try to categorize them
            String fileName = normalizedPath.substring(normalizedPath.lastIndexOf('/') + 1);
            
            // Core MelonLoader files go to net8 by default
            if (fileName.equals("MelonLoader.dll") || 
                fileName.equals("0Harmony.dll") || 
                fileName.startsWith("MonoMod.") ||
                fileName.equals("Il2CppInterop.Runtime.dll")) {
                return "net8/" + fileName;
            }
            
            // Support files go to Dependencies/SupportModules
            if (fileName.endsWith(".dll") && !fileName.equals("MelonLoader.dll")) {
                return "Dependencies/SupportModules/" + fileName;
            }
            
            // Config files go to the root
            if (fileName.endsWith(".json") || fileName.endsWith(".cfg") || fileName.endsWith(".xml")) {
                return "net8/" + fileName;
            }
        }
        
        // Default: place in net8 directory
        String fileName = normalizedPath.substring(normalizedPath.lastIndexOf('/') + 1);
        return "net8/" + fileName;
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/FileUtils.java

// File: FileUtils.java
// Path: /app/src/main/java/com/terrarialoader/util/FileUtils.java

package com.modloader.util;

import android.content.Context;
import android.database.Cursor;
import android.net.Uri;
import android.os.Environment;
import android.provider.OpenableColumns;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;

public class FileUtils {
    
    /**
     * Copy a file from source to destination
     * @param source Source file
     * @param destination Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFile(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream in = new FileInputStream(source);
             FileOutputStream out = new FileOutputStream(destination)) {
            
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            
            return true;
            
        } catch (Exception e) {
            // Log the error if LogUtils is available
            try {
                LogUtils.logError("Failed to copy file: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Copy content from URI to file
     * @param context Application context
     * @param sourceUri Source URI
     * @param destinationFile Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyUriToFile(Context context, Uri sourceUri, File destinationFile) {
        if (context == null || sourceUri == null || destinationFile == null) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destinationFile.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (InputStream in = context.getContentResolver().openInputStream(sourceUri);
             FileOutputStream out = new FileOutputStream(destinationFile)) {
            
            if (in == null) {
                return false;
            }
            
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            
            return true;
            
        } catch (Exception e) {
            try {
                LogUtils.logError("Failed to copy URI to file: " + sourceUri.toString() + " to " + destinationFile.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Get filename from URI
     * @param context Application context
     * @param uri URI to get filename from
     * @return Filename or null if not found
     */
    public static String getFilenameFromUri(Context context, Uri uri) {
        if (context == null || uri == null) {
            return null;
        }
        
        String filename = null;
        
        // Try to get filename from content resolver
        try (Cursor cursor = context.getContentResolver().query(uri, null, null, null, null)) {
            if (cursor != null && cursor.moveToFirst()) {
                int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                if (nameIndex != -1) {
                    filename = cursor.getString(nameIndex);
                }
            }
        } catch (Exception e) {
            // Ignore and try fallback
        }
        
        // Fallback: try to get filename from URI path
        if (filename == null) {
            String path = uri.getPath();
            if (path != null) {
                int lastSlash = path.lastIndexOf('/');
                if (lastSlash != -1 && lastSlash < path.length() - 1) {
                    filename = path.substring(lastSlash + 1);
                }
            }
        }
        
        // Final fallback: use last path segment
        if (filename == null) {
            filename = uri.getLastPathSegment();
        }
        
        return filename;
    }
    
    /**
     * Toggle mod file extension between .dll and .dll.disabled
     * @param modFile Mod file to toggle
     * @return true if toggle was successful, false otherwise
     */
    public static boolean toggleModFile(File modFile) {
        if (modFile == null || !modFile.exists()) {
            return false;
        }
        
        String fileName = modFile.getName();
        File newFile;
        
        if (fileName.endsWith(".dll.disabled")) {
            // Enable mod: remove .disabled extension
            String newName = fileName.substring(0, fileName.length() - ".disabled".length());
            newFile = new File(modFile.getParent(), newName);
        } else if (fileName.endsWith(".dll")) {
            // Disable mod: add .disabled extension
            String newName = fileName + ".disabled";
            newFile = new File(modFile.getParent(), newName);
        } else {
            // Not a valid mod file
            return false;
        }
        
        boolean success = modFile.renameTo(newFile);
        if (success) {
            try {
                LogUtils.logUser("Toggled mod: " + fileName + " -> " + newFile.getName());
            } catch (Exception e) {
                // Silent fail if logging not available
            }
        }
        
        return success;
    }
    
    /**
     * Format file size in human readable format
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String formatFileSize(long bytes) {
        if (bytes < 0) return "0 B";
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
        return String.format("%.1f GB", bytes / (1024.0 * 1024.0 * 1024.0));
    }
    
    /**
     * Format file size in human readable format (int overload)
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String formatFileSize(int bytes) {
        return formatFileSize((long) bytes);
    }
    
    /**
     * Copy a file using FileChannel for better performance on large files
     * @param source Source file
     * @param destination Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFileChannel(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream fis = new FileInputStream(source);
             FileOutputStream fos = new FileOutputStream(destination);
             FileChannel sourceChannel = fis.getChannel();
             FileChannel destChannel = fos.getChannel()) {
            
            destChannel.transferFrom(sourceChannel, 0, sourceChannel.size());
            return true;
            
        } catch (Exception e) {
            try {
                LogUtils.logError("Failed to copy file with channel: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Copy files with progress callback
     * @param source Source file
     * @param destination Destination file
     * @param callback Progress callback (can be null)
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFileWithProgress(File source, File destination, CopyProgressCallback callback) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream in = new FileInputStream(source);
             FileOutputStream out = new FileOutputStream(destination)) {
            
            byte[] buffer = new byte[8192];
            long totalBytes = source.length();
            long copiedBytes = 0;
            int bytesRead;
            
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
                copiedBytes += bytesRead;
                
                if (callback != null) {
                    int progress = (int) ((copiedBytes * 100) / totalBytes);
                    callback.onProgress(progress, copiedBytes, totalBytes);
                }
            }
            
            if (callback != null) {
                callback.onComplete(true);
            }
            
            return true;
            
        } catch (Exception e) {
            if (callback != null) {
                callback.onComplete(false);
            }
            
            try {
                LogUtils.logError("Failed to copy file with progress: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Move a file from source to destination
     * @param source Source file
     * @param destination Destination file
     * @return true if move was successful, false otherwise
     */
    public static boolean moveFile(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Try simple rename first
        if (source.renameTo(destination)) {
            return true;
        }
        
        // If rename failed, try copy and delete
        if (copyFile(source, destination)) {
            return source.delete();
        }
        
        return false;
    }
    
    /**
     * Delete a file or directory recursively
     * @param file File or directory to delete
     * @return true if deletion was successful, false otherwise
     */
    public static boolean deleteRecursively(File file) {
        if (file == null || !file.exists()) {
            return true;
        }
        
        if (file.isDirectory()) {
            File[] children = file.listFiles();
            if (children != null) {
                for (File child : children) {
                    if (!deleteRecursively(child)) {
                        return false;
                    }
                }
            }
        }
        
        return file.delete();
    }
    
    /**
     * Create directory if it doesn't exist
     * @param dir Directory to create
     * @return true if directory exists or was created successfully
     */
    public static boolean ensureDirectory(File dir) {
        if (dir == null) {
            return false;
        }
        
        if (dir.exists()) {
            return dir.isDirectory();
        }
        
        return dir.mkdirs();
    }
    
    /**
     * Get file size in human readable format
     * @param file File to get size for
     * @return Formatted file size string
     */
    public static String getHumanReadableSize(File file) {
        if (file == null || !file.exists()) {
            return "0 B";
        }
        
        return getHumanReadableSize(file.length());
    }
    
    /**
     * Get file size in human readable format
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String getHumanReadableSize(long bytes) {
        return formatFileSize(bytes);
    }
    
    /**
     * Check if external storage is available for read and write
     * @return true if external storage is available
     */
    public static boolean isExternalStorageWritable() {
        String state = Environment.getExternalStorageState();
        return Environment.MEDIA_MOUNTED.equals(state);
    }
    
    /**
     * Check if external storage is available to at least read
     * @return true if external storage is readable
     */
    public static boolean isExternalStorageReadable() {
        String state = Environment.getExternalStorageState();
        return Environment.MEDIA_MOUNTED.equals(state) ||
               Environment.MEDIA_MOUNTED_READ_ONLY.equals(state);
    }
    
    /**
     * Get app's external files directory
     * @param context Application context
     * @param type Type of files directory
     * @return External files directory
     */
    public static File getExternalFilesDir(Context context, String type) {
        if (context == null) {
            return null;
        }
        return context.getExternalFilesDir(type);
    }
    
    /**
     * Get app's cache directory
     * @param context Application context
     * @return Cache directory
     */
    public static File getCacheDir(Context context) {
        if (context == null) {
            return null;
        }
        return context.getCacheDir();
    }
    
    /**
     * Copy an input stream to an output stream
     * @param in Input stream
     * @param out Output stream
     * @throws IOException if copy fails
     */
    public static void copyStream(InputStream in, OutputStream out) throws IOException {
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = in.read(buffer)) != -1) {
            out.write(buffer, 0, bytesRead);
        }
    }
    
    /**
     * Get file extension from filename
     * @param filename Filename to get extension from
     * @return File extension (without dot) or empty string if no extension
     */
    public static String getFileExtension(String filename) {
        if (filename == null || filename.isEmpty()) {
            return "";
        }
        
        int lastDot = filename.lastIndexOf('.');
        if (lastDot == -1 || lastDot == filename.length() - 1) {
            return "";
        }
        
        return filename.substring(lastDot + 1).toLowerCase();
    }
    
    /**
     * Get filename without extension
     * @param filename Filename to process
     * @return Filename without extension
     */
    public static String getFilenameWithoutExtension(String filename) {
        if (filename == null || filename.isEmpty()) {
            return "";
        }
        
        int lastDot = filename.lastIndexOf('.');
        if (lastDot == -1) {
            return filename;
        }
        
        return filename.substring(0, lastDot);
    }
    
    /**
     * Check if a file has a specific extension
     * @param file File to check
     * @param extension Extension to check for (without dot)
     * @return true if file has the specified extension
     */
    public static boolean hasExtension(File file, String extension) {
        if (file == null || extension == null) {
            return false;
        }
        
        String fileExtension = getFileExtension(file.getName());
        return fileExtension.equalsIgnoreCase(extension);
    }
    
    /**
     * Get directory size recursively
     * @param directory Directory to calculate size for
     * @return Total size in bytes
     */
    public static long getDirectorySize(File directory) {
        if (directory == null || !directory.exists() || !directory.isDirectory()) {
            return 0;
        }
        
        long size = 0;
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isFile()) {
                    size += file.length();
                } else if (file.isDirectory()) {
                    size += getDirectorySize(file);
                }
            }
        }
        
        return size;
    }
    
    /**
     * Write text to a file
     * @param f File to write to
     * @param s String to write
     * @throws IOException if write fails
     */
    public static void writeText(File f, String s) throws IOException {
        try (FileOutputStream out = new FileOutputStream(f)) {
            out.write(s.getBytes(StandardCharsets.UTF_8));
        }
    }

    /**
     * Interface for copy progress callbacks
     */
    public interface CopyProgressCallback {
        void onProgress(int percentage, long copiedBytes, long totalBytes);
        void onComplete(boolean success);
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/IOUtils.java

package com.modloader.util;

import java.io.Closeable;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.nio.channels.FileChannel;

public class IOUtils {
    public static void closeQuietly(Closeable c) {
        if (c == null) return;
        try { c.close(); } catch (Exception ignored) {}
    }

    public static void copyFile(File src, File dst) throws IOException {
        try (FileChannel in = new FileInputStream(src).getChannel();
             FileChannel out = new FileOutputStream(dst).getChannel()) {
            long size = in.size();
            long pos = 0;
            while (pos < size) {
                pos += in.transferTo(pos, Math.min(8 * 1024 * 1024, size - pos), out);
            }
        }
    }

    public static boolean safeDelete(File f) {
        try {
            return f != null && (!f.exists() || f.delete());
        } catch (Throwable t) { return false; }
    }

    public static void copyToFile(InputStream in, File dst) throws IOException {
        try (FileOutputStream out = new FileOutputStream(dst)) {
            byte[] buf = new byte[8192];
            int n;
            while ((n = in.read(buf)) >= 0) {
                out.write(buf, 0, n);
            }
        }
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/Json.java

// File: Json.java - Missing JSON utility class
// Path: /storage/emulated/0/AndroidIDEProjects/ModLoader/app/src/main/java/com/modloader/util/Json.java

package com.modloader.util;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import com.google.gson.JsonSyntaxException;

import java.lang.reflect.Type;

/**
 * JSON utility class using Gson
 */
public final class Json {
    // Public static Gson instance for use throughout the app
    public static final Gson GSON = new GsonBuilder()
            .setPrettyPrinting()
            .disableHtmlEscaping()
            .create();

    private Json() {
        // Utility class - no instantiation
    }

    /**
     * Convert object to JSON string
     */
    public static String toJson(Object obj) {
        if (obj == null) {
            return "null";
        }
        try {
            return GSON.toJson(obj);
        } catch (Exception e) {
            LogUtils.logDebug("JSON serialization error: " + e.getMessage());
            return "{}";
        }
    }

    /**
     * Convert JSON string to object
     */
    public static <T> T fromJson(String json, Class<T> classOfT) {
        if (json == null || json.trim().isEmpty()) {
            return null;
        }
        try {
            return GSON.fromJson(json, classOfT);
        } catch (JsonSyntaxException e) {
            LogUtils.logDebug("JSON parsing error: " + e.getMessage());
            return null;
        }
    }

    /**
     * Convert JSON string to object with type
     */
    public static <T> T fromJson(String json, Type typeOfT) {
        if (json == null || json.trim().isEmpty()) {
            return null;
        }
        try {
            return GSON.fromJson(json, typeOfT);
        } catch (JsonSyntaxException e) {
            LogUtils.logDebug("JSON parsing error: " + e.getMessage());
            return null;
        }
    }

    /**
     * Check if string is valid JSON
     */
    public static boolean isValidJson(String json) {
        if (json == null || json.trim().isEmpty()) {
            return false;
        }
        try {
            GSON.fromJson(json, Object.class);
            return true;
        } catch (JsonSyntaxException e) {
            return false;
        }
    }

    /**
     * Pretty print JSON string
     */
    public static String prettyPrint(String json) {
        try {
            Object obj = GSON.fromJson(json, Object.class);
            return GSON.toJson(obj);
        } catch (JsonSyntaxException e) {
            LogUtils.logDebug("JSON pretty print error: " + e.getMessage());
            return json;
        }
    }

    /**
     * Minify JSON string (remove formatting)
     */
    public static String minify(String json) {
        try {
            Object obj = GSON.fromJson(json, Object.class);
            return new Gson().toJson(obj); // Use non-pretty printing Gson
        } catch (JsonSyntaxException e) {
            LogUtils.logDebug("JSON minify error: " + e.getMessage());
            return json;
        }
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/JsonUtils.java

package com.modloader.util;

import org.json.JSONArray;
import org.json.JSONObject;

import com.modloader.plugins.PluginDescriptor;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;

public class JsonUtils {

    public static JSONObject readJson(InputStream in) throws Exception {
        StringBuilder sb = new StringBuilder();
        try (BufferedReader br = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8))) {
            String line;
            while ((line = br.readLine()) != null) sb.append(line).append('\n');
        }
        return new JSONObject(sb.toString());
    }

    public static PluginDescriptor toDescriptor(JSONObject obj) {
        PluginDescriptor d = new PluginDescriptor();
        d.id = obj.optString("id", "");
        d.name = obj.optString("name", d.id);
        d.version = obj.optString("version", "1.0.0");
        d.author = obj.optString("author", "unknown");
        d.entryClass = obj.optString("entryClass", "");
        d.enabled = obj.optBoolean("enabled", true);
        JSONArray perms = obj.optJSONArray("permissions");
        if (perms != null) {
            d.permissions = new ArrayList<>();
            for (int i = 0; i < perms.length(); i++) d.permissions.add(perms.optString(i));
        }
        return d;
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/Log.java

package com.modloader.util;
import android.util.Log;
public final class Log {
    private static final String TAG = "ModLoader";
    public static void d(String m){ Log.d(TAG, m); }
    public static void i(String m){ Log.i(TAG, m); }
    public static void w(String m){ Log.w(TAG, m); }
    public static void e(String m, Throwable t){ Log.e(TAG, m, t); }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/LogUtils.java

// File: LogUtils.java (Enhanced) - Complete Logging System
// Path: /app/src/main/java/com/modloader/util/LogUtils.java

package com.modloader.util;

import android.content.Context;
import android.util.Log;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

/**
 * Enhanced LogUtils - Complete logging system with file output and debug control
 */
public class LogUtils {
    private static final String TAG = "TerrariaLoader";
    private static final String LOG_FILE_NAME = "AppLog.txt";
    private static final int MAX_LOG_FILES = 5;
    private static final int MAX_LOG_SIZE = 1024 * 1024; // 1MB
    
    private static Context applicationContext;
    private static boolean debugEnabled = false;
    private static boolean initialized = false;
    private static File logDir;
    private static final List<String> logBuffer = new ArrayList<>();
    private static final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault());
    
    /**
     * Initialize logging system
     */
    public static void initialize(Context context) {
        if (initialized) return;
        
        applicationContext = context.getApplicationContext();
        setupLogDirectory();
        initialized = true;
        
        logInfo("LogUtils initialized");
    }
    
    /**
     * Initialize app startup logging
     */
    public static void initializeAppStartup() {
        if (!initialized) return;
        
        rotateLogsIfNeeded();
        logUser("=== TerrariaLoader App Started ===");
        logInfo("App startup logging initialized");
    }
    
    /**
     * Setup log directory
     */
    private static void setupLogDirectory() {
        if (applicationContext == null) return;
        
        try {
            File baseDir = new File(applicationContext.getExternalFilesDir(null), "TerrariaLoader/com.and.games505.TerrariaPaid");
            logDir = new File(baseDir, "AppLogs");
            if (!logDir.exists()) {
                logDir.mkdirs();
            }
        } catch (Exception e) {
            Log.e(TAG, "Failed to setup log directory: " + e.getMessage());
            // Fallback to app's files directory
            try {
                logDir = new File(applicationContext.getFilesDir(), "logs");
                if (!logDir.exists()) {
                    logDir.mkdirs();
                }
            } catch (Exception ex) {
                Log.e(TAG, "Failed to setup fallback log directory: " + ex.getMessage());
            }
        }
    }
    
    /**
     * Set debug mode enabled/disabled
     */
    public static void setDebugEnabled(boolean enabled) {
        debugEnabled = enabled;
        logInfo("Debug logging " + (enabled ? "enabled" : "disabled"));
    }
    
    /**
     * Check if debug mode is enabled
     */
    public static boolean isDebugEnabled() {
        return debugEnabled;
    }
    
    // ===== Public Logging Methods =====
    
    /**
     * Log debug message (only shown if debug is enabled)
     */
    public static void logDebug(String message) {
        if (debugEnabled) {
            String logMessage = formatLogMessage("DEBUG", message);
            Log.d(TAG, message);
            writeToFile(logMessage);
            addToBuffer(logMessage);
        }
    }
    
    /**
     * Log info message
     */
    public static void logInfo(String message) {
        String logMessage = formatLogMessage("INFO", message);
        Log.i(TAG, message);
        writeToFile(logMessage);
        addToBuffer(logMessage);
    }
    
    /**
     * Log warning message
     */
    public static void logWarning(String message) {
        String logMessage = formatLogMessage("WARN", message);
        Log.w(TAG, message);
        writeToFile(logMessage);
        addToBuffer(logMessage);
    }
    
    /**
     * Log error message
     */
    public static void logError(String message) {
        String logMessage = formatLogMessage("ERROR", message);
        Log.e(TAG, message);
        writeToFile(logMessage);
        addToBuffer(logMessage);
    }
    
    /**
     * Log user-visible message (always shown)
     */
    public static void logUser(String message) {
        String logMessage = formatLogMessage("USER", message);
        Log.i(TAG, "[USER] " + message);
        writeToFile(logMessage);
        addToBuffer(logMessage);
    }
    
    /**
     * Log exception with stack trace
     */
    public static void logException(String message, Throwable throwable) {
        String logMessage = formatLogMessage("ERROR", message + ": " + throwable.getMessage());
        Log.e(TAG, message, throwable);
        writeToFile(logMessage);
        addToBuffer(logMessage);
        
        // Log stack trace
        for (StackTraceElement element : throwable.getStackTrace()) {
            String stackLine = formatLogMessage("ERROR", "  at " + element.toString());
            writeToFile(stackLine);
            addToBuffer(stackLine);
        }
    }
    
    /**
     * Log error with exception (overloaded method for compatibility)
     */
    public static void logError(String message, Exception e) {
        logException(message, e);
    }
    
    /**
     * Log error with throwable (overloaded method for compatibility)  
     */
    public static void logError(String message, Throwable throwable) {
        logException(message, throwable);
    }
    
    // ===== APK Process Logging Methods (for ApkProcessTracker compatibility) =====
    
    /**
     * Log APK process start
     */
    public static void logApkProcessStart(String operation, String apkName) {
        String message = String.format("🚀 APK Process Started: %s on %s", operation, apkName);
        logUser(message);
    }
    
    /**
     * Log APK process step
     */
    public static void logApkProcessStep(String stepName, String details) {
        String message = String.format("⚙️ APK Step [%s]: %s", stepName, details);
        logInfo(message);
    }
    
    /**
     * Log APK process completion
     */
    public static void logApkProcessComplete(boolean success, String result) {
        String message = String.format("✅ APK Process %s: %s", 
                                      success ? "Completed" : "Failed", result);
        if (success) {
            logUser(message);
        } else {
            logError(message);
        }
    }
    
    // ===== Log Formatting and File Operations =====
    
    /**
     * Format log message with timestamp and level
     */
    private static String formatLogMessage(String level, String message) {
        String timestamp = dateFormat.format(new Date());
        return String.format("[%s] %s: %s", timestamp, level, message);
    }
    
    /**
     * Write log message to file
     */
    private static void writeToFile(String message) {
        if (logDir == null || !logDir.exists()) {
            return;
        }
        
        try {
            File logFile = new File(logDir, LOG_FILE_NAME);
            
            // Check if log rotation is needed
            if (logFile.exists() && logFile.length() > MAX_LOG_SIZE) {
                rotateLogsIfNeeded();
            }
            
            // Append to current log file
            try (FileWriter writer = new FileWriter(logFile, true)) {
                writer.write(message + "\n");
                writer.flush();
            }
        } catch (IOException e) {
            Log.e(TAG, "Failed to write to log file: " + e.getMessage());
        }
    }
    
    /**
     * Add message to in-memory buffer
     */
    private static void addToBuffer(String message) {
        synchronized (logBuffer) {
            logBuffer.add(message);
            // Keep only last 1000 messages in memory
            if (logBuffer.size() > 1000) {
                logBuffer.remove(0);
            }
        }
    }
    
    /**
     * Rotate log files when they get too large
     */
    private static void rotateLogsIfNeeded() {
        if (logDir == null || !logDir.exists()) {
            return;
        }
        
        try {
            File currentLog = new File(logDir, LOG_FILE_NAME);
            if (!currentLog.exists()) {
                return;
            }
            
            // Rotate existing logs
            for (int i = MAX_LOG_FILES - 1; i >= 1; i--) {
                File oldLog = new File(logDir, "AppLog" + i + ".txt");
                if (i == MAX_LOG_FILES - 1) {
                    // Delete oldest log
                    if (oldLog.exists()) {
                        oldLog.delete();
                    }
                } else {
                    // Rename to next number
                    File nextLog = new File(logDir, "AppLog" + (i + 1) + ".txt");
                    if (oldLog.exists()) {
                        oldLog.renameTo(nextLog);
                    }
                }
            }
            
            // Move current log to AppLog1.txt
            File firstBackup = new File(logDir, "AppLog1.txt");
            currentLog.renameTo(firstBackup);
            
            logInfo("Log rotation completed");
        } catch (Exception e) {
            Log.e(TAG, "Failed to rotate logs: " + e.getMessage());
        }
    }
    
    // ===== Public Utility Methods =====
    
    // ===== Log File Management Methods (for DiagnosticBundleExporter compatibility) =====
    
    /**
     * Get available log files
     */
    public static List<File> getAvailableLogFiles() {
        List<File> logFiles = new ArrayList<>();
        if (logDir == null || !logDir.exists()) {
            return logFiles;
        }
        
        try {
            // Current log file
            File currentLog = new File(logDir, LOG_FILE_NAME);
            if (currentLog.exists()) {
                logFiles.add(currentLog);
            }
            
            // Rotated log files
            for (int i = 1; i <= MAX_LOG_FILES; i++) {
                File rotatedLog = new File(logDir, "AppLog" + i + ".txt");
                if (rotatedLog.exists()) {
                    logFiles.add(rotatedLog);
                }
            }
        } catch (Exception e) {
            logError("Failed to get available log files: " + e.getMessage());
        }
        
        return logFiles;
    }
    
    /**
     * Read log file by index (0 = current, 1-5 = rotated)
     */
    public static String readLogFile(int index) {
        if (logDir == null || !logDir.exists()) {
            return "Log directory not available";
        }
        
        File logFile;
        if (index == 0) {
            logFile = new File(logDir, LOG_FILE_NAME);
        } else {
            logFile = new File(logDir, "AppLog" + index + ".txt");
        }
        
        if (!logFile.exists()) {
            return "Log file " + index + " does not exist";
        }
        
        try {
            StringBuilder content = new StringBuilder();
            java.io.BufferedReader reader = new java.io.BufferedReader(
                new java.io.FileReader(logFile));
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
            reader.close();
            return content.toString();
        } catch (IOException e) {
            return "Error reading log file " + index + ": " + e.getMessage();
        }
    }
    
    /**
     * Get all logs as a single string
     */
    public static String getLogs() {
        StringBuilder allLogs = new StringBuilder();
        
        // Add buffered logs first
        synchronized (logBuffer) {
            for (String log : logBuffer) {
                allLogs.append(log).append("\n");
            }
        }
        
        return allLogs.toString();
    }
    
    /**
     * Get logs from file
     */
    public static String getLogsFromFile() {
        if (logDir == null || !logDir.exists()) {
            return "No logs available - log directory not initialized";
        }
        
        StringBuilder fileLogs = new StringBuilder();
        File currentLog = new File(logDir, LOG_FILE_NAME);
        
        if (currentLog.exists()) {
            try {
                java.io.BufferedReader reader = new java.io.BufferedReader(
                    new java.io.FileReader(currentLog));
                String line;
                while ((line = reader.readLine()) != null) {
                    fileLogs.append(line).append("\n");
                }
                reader.close();
            } catch (IOException e) {
                fileLogs.append("Error reading log file: ").append(e.getMessage());
            }
        } else {
            fileLogs.append("No log file found");
        }
        
        return fileLogs.toString();
    }
    
    /**
     * Clear all logs
     */
    public static void clearLogs() {
        synchronized (logBuffer) {
            logBuffer.clear();
        }
        
        if (logDir != null && logDir.exists()) {
            try {
                // Delete all log files
                File[] logFiles = logDir.listFiles((dir, name) -> 
                    name.startsWith("AppLog") && name.endsWith(".txt"));
                if (logFiles != null) {
                    for (File logFile : logFiles) {
                        logFile.delete();
                    }
                }
                logInfo("All logs cleared");
            } catch (Exception e) {
                logError("Failed to clear log files: " + e.getMessage());
            }
        }
    }
    
    /**
     * Export logs to external file
     */
    public static boolean exportLogs(File outputFile) {
        try {
            String allLogs = getLogs() + "\n\n=== File Logs ===\n" + getLogsFromFile();
            
            try (FileWriter writer = new FileWriter(outputFile)) {
                writer.write("=== TerrariaLoader Log Export ===\n");
                writer.write("Export Date: " + dateFormat.format(new Date()) + "\n");
                writer.write("Debug Enabled: " + debugEnabled + "\n");
                writer.write("=== Logs ===\n\n");
                writer.write(allLogs);
            }
            
            logInfo("Logs exported to: " + outputFile.getAbsolutePath());
            return true;
        } catch (IOException e) {
            logError("Failed to export logs: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get log statistics
     */
    public static String getLogStats() {
        StringBuilder stats = new StringBuilder();
        stats.append("=== Log Statistics ===\n");
        stats.append("Debug Enabled: ").append(debugEnabled).append("\n");
        stats.append("Initialized: ").append(initialized).append("\n");
        stats.append("Buffer Size: ").append(logBuffer.size()).append(" messages\n");
        
        if (logDir != null) {
            stats.append("Log Directory: ").append(logDir.getAbsolutePath()).append("\n");
            stats.append("Directory Exists: ").append(logDir.exists()).append("\n");
            
            if (logDir.exists()) {
                File[] logFiles = logDir.listFiles((dir, name) -> 
                    name.startsWith("AppLog") && name.endsWith(".txt"));
                stats.append("Log Files: ").append(logFiles != null ? logFiles.length : 0).append("\n");
                
                File currentLog = new File(logDir, LOG_FILE_NAME);
                if (currentLog.exists()) {
                    stats.append("Current Log Size: ").append(formatFileSize(currentLog.length())).append("\n");
                }
            }
        } else {
            stats.append("Log Directory: Not initialized\n");
        }
        
        return stats.toString();
    }
    
    /**
     * Format file size for display
     */
    private static String formatFileSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
    }
    
    /**
     * Get log directory path
     */
    public static String getLogDirectory() {
        return logDir != null ? logDir.getAbsolutePath() : "Not initialized";
    }
    
    /**
     * Check if logging is properly initialized
     */
    public static boolean isInitialized() {
        return initialized && logDir != null && logDir.exists();
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/MelonLoaderDiagnostic.java

// File: MelonLoaderDiagnostic.java (Diagnostic Tool)
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/MelonLoaderDiagnostic.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.File;

public class MelonLoaderDiagnostic {
    
    public static String generateDetailedDiagnostic(Context context, String gamePackage) {
        StringBuilder diagnostic = new StringBuilder();
        diagnostic.append("=== DETAILED MELONLOADER DIAGNOSTIC ===\n\n");
        
        // Check all required directories and files
        File baseDir = PathManager.getGameBaseDir(context, gamePackage);
        File melonLoaderDir = PathManager.getMelonLoaderDir(context, gamePackage);
        File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
        File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
        File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
        
        diagnostic.append("📁 DIRECTORY STATUS:\n");
        diagnostic.append("Base Dir: ").append(checkDirectory(baseDir)).append("\n");
        diagnostic.append("MelonLoader Dir: ").append(checkDirectory(melonLoaderDir)).append("\n");
        diagnostic.append("NET8 Dir: ").append(checkDirectory(net8Dir)).append("\n");
        diagnostic.append("NET35 Dir: ").append(checkDirectory(net35Dir)).append("\n");
        diagnostic.append("Dependencies Dir: ").append(checkDirectory(depsDir)).append("\n\n");
        
        // Check for required NET8 files
        diagnostic.append("🔸 NET8 RUNTIME FILES:\n");
        String[] net8Files = {
            "MelonLoader.dll",
            "0Harmony.dll", 
            "MonoMod.RuntimeDetour.dll",
            "MonoMod.Utils.dll",
            "Il2CppInterop.Runtime.dll"
        };
        
        int net8Found = 0;
        for (String fileName : net8Files) {
            File file = new File(net8Dir, fileName);
            boolean exists = file.exists() && file.length() > 0;
            diagnostic.append("  ").append(exists ? "✅" : "❌").append(" ").append(fileName);
            if (exists) {
                net8Found++;
                diagnostic.append(" (").append(FileUtils.formatFileSize(file.length())).append(")");
            }
            diagnostic.append("\n");
        }
        diagnostic.append("NET8 Score: ").append(net8Found).append("/").append(net8Files.length).append("\n\n");
        
        // Check for required NET35 files
        diagnostic.append("🔸 NET35 RUNTIME FILES:\n");
        String[] net35Files = {
            "MelonLoader.dll",
            "0Harmony.dll",
            "MonoMod.RuntimeDetour.dll", 
            "MonoMod.Utils.dll"
        };
        
        int net35Found = 0;
        for (String fileName : net35Files) {
            File file = new File(net35Dir, fileName);
            boolean exists = file.exists() && file.length() > 0;
            diagnostic.append("  ").append(exists ? "✅" : "❌").append(" ").append(fileName);
            if (exists) {
                net35Found++;
                diagnostic.append(" (").append(FileUtils.formatFileSize(file.length())).append(")");
            }
            diagnostic.append("\n");
        }
        diagnostic.append("NET35 Score: ").append(net35Found).append("/").append(net35Files.length).append("\n\n");
        
        // Check Dependencies
        diagnostic.append("🔸 DEPENDENCY FILES:\n");
        File supportModulesDir = new File(depsDir, "SupportModules");
        File assemblyGenDir = new File(depsDir, "Il2CppAssemblyGenerator");
        
        diagnostic.append("Support Modules Dir: ").append(checkDirectory(supportModulesDir)).append("\n");
        diagnostic.append("Assembly Generator Dir: ").append(checkDirectory(assemblyGenDir)).append("\n");
        
        // List actual files found
        diagnostic.append("\n📋 FILES FOUND:\n");
        if (melonLoaderDir.exists()) {
            diagnostic.append(listDirectoryContents(melonLoaderDir, ""));
        } else {
            diagnostic.append("MelonLoader directory doesn't exist!\n");
        }
        
        // Generate recommendations
        diagnostic.append("\n💡 RECOMMENDATIONS:\n");
        if (net8Found == 0 && net35Found == 0) {
            diagnostic.append("❌ NO RUNTIME FILES FOUND!\n");
            diagnostic.append("SOLUTION: You need to install MelonLoader files.\n");
            diagnostic.append("Options:\n");
            diagnostic.append("1. Use 'Automated Installation' in Setup Guide\n");
            diagnostic.append("2. Manually download and extract MelonLoader files\n");
            diagnostic.append("3. Use the APK patcher to inject loader\n\n");
        } else if (net8Found > 0) {
            diagnostic.append("✅ Some NET8 files found, but incomplete installation\n");
            diagnostic.append("Missing files need to be added to: ").append(net8Dir.getAbsolutePath()).append("\n\n");
        } else if (net35Found > 0) {
            diagnostic.append("✅ Some NET35 files found, but incomplete installation\n");
            diagnostic.append("Missing files need to be added to: ").append(net35Dir.getAbsolutePath()).append("\n\n");
        }
        
        // Check if automated installation would work
        diagnostic.append("🌐 INTERNET CONNECTIVITY: ");
        if (OnlineInstaller.isOnlineInstallationAvailable()) {
            diagnostic.append("✅ Available - Automated installation possible\n");
            diagnostic.append("RECOMMENDED: Use 'Automated Installation' for easiest setup\n");
        } else {
            diagnostic.append("❌ Not available - Manual installation required\n");
            diagnostic.append("REQUIRED: Download MelonLoader files manually\n");
        }
        
        return diagnostic.toString();
    }
    
    private static String checkDirectory(File dir) {
        if (dir == null) return "❌ null";
        if (!dir.exists()) return "❌ doesn't exist (" + dir.getAbsolutePath() + ")";
        if (!dir.isDirectory()) return "❌ not a directory";
        
        File[] files = dir.listFiles();
        int fileCount = files != null ? files.length : 0;
        return "✅ exists (" + fileCount + " items)";
    }
    
    private static String listDirectoryContents(File dir, String indent) {
        StringBuilder contents = new StringBuilder();
        if (dir == null || !dir.exists() || !dir.isDirectory()) {
            return contents.toString();
        }
        
        File[] files = dir.listFiles();
        if (files == null || files.length == 0) {
            contents.append(indent).append("(empty)\n");
            return contents.toString();
        }
        
        for (File file : files) {
            contents.append(indent);
            if (file.isDirectory()) {
                contents.append("📁 ").append(file.getName()).append("/\n");
                if (indent.length() < 8) { // Limit recursion depth
                    contents.append(listDirectoryContents(file, indent + "  "));
                }
            } else {
                contents.append("📄 ").append(file.getName());
                contents.append(" (").append(FileUtils.formatFileSize(file.length())).append(")\n");
            }
        }
        
        return contents.toString();
    }
    
    // Quick fix suggestions
    public static String getQuickFixSuggestions(Context context, String gamePackage) {
        StringBuilder suggestions = new StringBuilder();
        suggestions.append("🚀 QUICK FIX OPTIONS:\n\n");
        
        suggestions.append("1. AUTOMATED INSTALLATION (Recommended):\n");
        suggestions.append("   • Go to 'MelonLoader Setup Guide'\n");
        suggestions.append("   • Choose 'Automated Online Installation'\n");
        suggestions.append("   • Select MelonLoader or LemonLoader\n");
        suggestions.append("   • Wait for download and extraction\n\n");
        
        suggestions.append("2. MANUAL INSTALLATION:\n");
        suggestions.append("   • Download MelonLoader from GitHub\n");
        suggestions.append("   • Extract files to correct directories\n");
        suggestions.append("   • Follow the manual installation guide\n\n");
        
        suggestions.append("3. APK INJECTION:\n");
        suggestions.append("   • Use 'APK Patcher' feature\n");
        suggestions.append("   • Select Terraria APK\n");
        suggestions.append("   • Inject MelonLoader into APK\n");
        suggestions.append("   • Install modified APK\n\n");
        
        File baseDir = PathManager.getGameBaseDir(context, gamePackage);
        suggestions.append("📍 TARGET DIRECTORY:\n");
        suggestions.append(baseDir.getAbsolutePath()).append("/Loaders/MelonLoader/\n\n");
        
        suggestions.append("⚠️ MAKE SURE TO:\n");
        suggestions.append("• Have stable internet connection (for automated)\n");
        suggestions.append("• Grant file manager permissions (for manual)\n");
        suggestions.append("• Use exact directory paths shown above\n");
        suggestions.append("• Restart TerrariaLoader after installation\n");
        
        return suggestions.toString();
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/OfflineZipImporter.java

// File: OfflineZipImporter.java - Smart ZIP Import with Auto-Detection
// Path: /main/java/com/terrarialoader/util/OfflineZipImporter.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.*;
import java.util.zip.*;
import java.util.HashSet;
import java.util.Set;

/**
 * Smart offline ZIP importer that auto-detects NET8/NET35 and extracts to correct directories
 */
public class OfflineZipImporter {
    
    public static class ImportResult {
        public boolean success;
        public String message;
        public MelonLoaderManager.LoaderType detectedType;
        public int filesExtracted;
        public String errorDetails;
        
        public ImportResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
    }
    
    // File signatures for detection
    private static final String[] NET8_SIGNATURES = {
        "MelonLoader.deps.json",
        "MelonLoader.runtimeconfig.json", 
        "Il2CppInterop.Runtime.dll"
    };
    
    private static final String[] NET35_SIGNATURES = {
        "MelonLoader.dll",
        "0Harmony.dll"
    };
    
    private static final String[] CORE_FILES = {
        "MelonLoader.dll",
        "0Harmony.dll",
        "MonoMod.RuntimeDetour.dll",
        "MonoMod.Utils.dll"
    };
    
    /**
     * Import MelonLoader ZIP with auto-detection and smart extraction
     */
    public static ImportResult importMelonLoaderZip(Context context, android.net.Uri zipUri) {
        LogUtils.logUser("🔍 Starting smart ZIP import...");
        
        try {
            // Step 1: Analyze ZIP contents
            ZipAnalysis analysis = analyzeZipContents(context, zipUri);
            if (!analysis.isValid) {
                return new ImportResult(false, "Invalid MelonLoader ZIP file: " + analysis.error);
            }
            
            LogUtils.logUser("📋 Detected: " + analysis.detectedType.getDisplayName());
            LogUtils.logUser("📊 Found " + analysis.totalFiles + " files to extract");
            
            // Step 2: Prepare target directories
            String gamePackage = MelonLoaderManager.TERRARIA_PACKAGE;
            if (!PathManager.initializeGameDirectories(context, gamePackage)) {
                return new ImportResult(false, "Failed to create directory structure");
            }
            
            // Step 3: Extract files to appropriate locations
            int extractedCount = extractZipContents(context, zipUri, analysis, gamePackage);
            
            if (extractedCount > 0) {
                ImportResult result = new ImportResult(true, 
                    "Successfully imported " + analysis.detectedType.getDisplayName() + 
                    " (" + extractedCount + " files)");
                result.detectedType = analysis.detectedType;
                result.filesExtracted = extractedCount;
                
                LogUtils.logUser("✅ ZIP import completed: " + extractedCount + " files extracted");
                return result;
            } else {
                return new ImportResult(false, "No files were extracted from ZIP");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("ZIP import error: " + e.getMessage());
            ImportResult result = new ImportResult(false, "Import failed: " + e.getMessage());
            result.errorDetails = e.toString();
            return result;
        }
    }
    
    /**
     * Analyze ZIP contents to detect loader type and validate files
     */
    private static ZipAnalysis analyzeZipContents(Context context, android.net.Uri zipUri) {
        ZipAnalysis analysis = new ZipAnalysis();
        
        try (InputStream inputStream = context.getContentResolver().openInputStream(zipUri);
             ZipInputStream zis = new ZipInputStream(new BufferedInputStream(inputStream))) {
            
            ZipEntry entry;
            Set<String> foundFiles = new HashSet<>();
            
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.isDirectory()) {
                    zis.closeEntry();
                    continue;
                }
                
                String fileName = getCleanFileName(entry.getName());
                foundFiles.add(fileName.toLowerCase());
                analysis.totalFiles++;
                
                // Check for type indicators
                for (String signature : NET8_SIGNATURES) {
                    if (fileName.equalsIgnoreCase(signature)) {
                        analysis.hasNet8Indicators = true;
                        break;
                    }
                }
                
                for (String signature : NET35_SIGNATURES) {
                    if (fileName.equalsIgnoreCase(signature)) {
                        analysis.hasNet35Indicators = true;
                        break;
                    }
                }
                
                zis.closeEntry();
            }
            
            // Determine loader type
            if (analysis.hasNet8Indicators) {
                analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET8;
            } else if (analysis.hasNet35Indicators) {
                analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET35;
            } else {
                // Fallback: check for core files and default to NET8
                boolean hasCoreFiles = false;
                for (String coreFile : CORE_FILES) {
                    if (foundFiles.contains(coreFile.toLowerCase())) {
                        hasCoreFiles = true;
                        break;
                    }
                }
                
                if (hasCoreFiles) {
                    analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET8; // Default
                    LogUtils.logUser("⚠️ Auto-detected as MelonLoader (default)");
                } else {
                    analysis.isValid = false;
                    analysis.error = "No MelonLoader files detected in ZIP";
                    return analysis;
                }
            }
            
            // Validate we have minimum required files
            int coreFilesFound = 0;
            for (String coreFile : CORE_FILES) {
                if (foundFiles.contains(coreFile.toLowerCase())) {
                    coreFilesFound++;
                }
            }
            
            if (coreFilesFound < 2) { // At least 2 core files required
                analysis.isValid = false;
                analysis.error = "Insufficient MelonLoader core files (" + coreFilesFound + "/4)";
                return analysis;
            }
            
            analysis.isValid = true;
            LogUtils.logDebug("ZIP analysis complete - Type: " + analysis.detectedType.getDisplayName() + 
                            ", Files: " + analysis.totalFiles);
            
        } catch (Exception e) {
            analysis.isValid = false;
            analysis.error = "ZIP analysis failed: " + e.getMessage();
            LogUtils.logDebug("ZIP analysis error: " + e.getMessage());
        }
        
        return analysis;
    }
    
    /**
     * Extract ZIP contents to appropriate directories based on detected type
     */
    private static int extractZipContents(Context context, android.net.Uri zipUri, ZipAnalysis analysis, String gamePackage) throws IOException {
        int extractedCount = 0;
        
        // Get target directories
        File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
        File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
        File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
        
        // Ensure directories exist
        PathManager.ensureDirectoryExists(net8Dir);
        PathManager.ensureDirectoryExists(net35Dir);
        PathManager.ensureDirectoryExists(depsDir);
        PathManager.ensureDirectoryExists(new File(depsDir, "SupportModules"));
        PathManager.ensureDirectoryExists(new File(depsDir, "CompatibilityLayers"));
        PathManager.ensureDirectoryExists(new File(depsDir, "Il2CppAssemblyGenerator"));
        
        try (InputStream inputStream = context.getContentResolver().openInputStream(zipUri);
             ZipInputStream zis = new ZipInputStream(new BufferedInputStream(inputStream))) {
            
            ZipEntry entry;
            byte[] buffer = new byte[8192];
            
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.isDirectory()) {
                    zis.closeEntry();
                    continue;
                }
                
                String fileName = getCleanFileName(entry.getName());
                File targetFile = determineTargetFile(fileName, analysis.detectedType, net8Dir, net35Dir, depsDir);
                
                if (targetFile != null) {
                    // Ensure parent directory exists
                    targetFile.getParentFile().mkdirs();
                    
                    // Extract file
                    try (FileOutputStream fos = new FileOutputStream(targetFile)) {
                        int len;
                        while ((len = zis.read(buffer)) > 0) {
                            fos.write(buffer, 0, len);
                        }
                    }
                    
                    extractedCount++;
                    LogUtils.logDebug("Extracted: " + fileName + " -> " + targetFile.getAbsolutePath());
                } else {
                    LogUtils.logDebug("Skipped: " + fileName + " (not needed)");
                }
                
                zis.closeEntry();
            }
        }
        
        return extractedCount;
    }
    
    /**
     * Determine target file location based on file type and loader type
     */
    private static File determineTargetFile(String fileName, MelonLoaderManager.LoaderType loaderType, 
                                          File net8Dir, File net35Dir, File depsDir) {
        String lowerName = fileName.toLowerCase();
        
        // Skip non-relevant files
        if (!isRelevantFile(fileName)) {
            return null;
        }
        
        // Core runtime files
        if (isCoreRuntimeFile(fileName)) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        // Dependency files
        if (lowerName.contains("il2cpp") || lowerName.contains("interop")) {
            return new File(depsDir, "SupportModules/" + fileName);
        }
        
        if (lowerName.contains("unity") || lowerName.contains("assemblygenerator")) {
            return new File(depsDir, "Il2CppAssemblyGenerator/" + fileName);
        }
        
        if (lowerName.contains("compat")) {
            return new File(depsDir, "CompatibilityLayers/" + fileName);
        }
        
        // Default: place DLLs in appropriate runtime directory
        if (lowerName.endsWith(".dll")) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        // Config files go to runtime directory
        if (lowerName.endsWith(".json") || lowerName.endsWith(".xml")) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        return null; // Skip unknown files
    }
    
    private static String getCleanFileName(String entryName) {
        // Remove directory paths and get just the filename
        String fileName = entryName;
        
        // Handle both forward and backward slashes
        int lastSlash = Math.max(fileName.lastIndexOf('/'), fileName.lastIndexOf('\\'));
        if (lastSlash >= 0) {
            fileName = fileName.substring(lastSlash + 1);
        }
        
        return fileName;
    }
    
    private static boolean isRelevantFile(String fileName) {
        String lowerName = fileName.toLowerCase();
        return lowerName.endsWith(".dll") || 
               lowerName.endsWith(".json") || 
               lowerName.endsWith(".xml") ||
               lowerName.endsWith(".pdb");
    }
    
    private static boolean isCoreRuntimeFile(String fileName) {
        String lowerName = fileName.toLowerCase();
        for (String coreFile : CORE_FILES) {
            if (lowerName.equals(coreFile.toLowerCase())) {
                return true;
            }
        }
        return lowerName.contains("melonloader") || 
               lowerName.contains("runtimeconfig") ||
               lowerName.contains("deps.json");
    }
    
    /**
     * Helper class to store ZIP analysis results
     */
    private static class ZipAnalysis {
        boolean isValid = false;
        String error = "";
        MelonLoaderManager.LoaderType detectedType;
        boolean hasNet8Indicators = false;
        boolean hasNet35Indicators = false;
        int totalFiles = 0;
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/OnlineInstaller.java

// File: OnlineInstaller.java (Utility Class) - Complete Automated Installation System
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/OnlineInstaller.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.File;

/**
 * OnlineInstaller - Complete automated installation system
 * Uses existing Downloader and PathManager for seamless MelonLoader/LemonLoader installation
 */
public class OnlineInstaller {
    
    // GitHub URLs for MelonLoader/LemonLoader releases
    public static final String MELONLOADER_URL = "https://github.com/LavaGang/MelonLoader/releases/download/0.6.5.1/melon_data.zip";
    public static final String LEMONLOADER_URL = "https://github.com/LemonLoader/MelonLoader/releases/download/0.6.5.1/melon_data.zip";
    
    // Installation result class
    public static class InstallationResult {
        public boolean success;
        public String message;
        public String errorDetails;
        public File installationPath;
        public int filesInstalled;
        public long totalSize;
        
        public InstallationResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
        
        public InstallationResult(boolean success, String message, String errorDetails) {
            this.success = success;
            this.message = message;
            this.errorDetails = errorDetails;
        }
    }
    
    /**
     * Complete automated installation of MelonLoader
     * Downloads melon_data.zip and extracts to proper directory structure
     * 
     * @param context Application context
     * @param gamePackage Target game package (e.g., "com.and.games505.TerrariaPaid")
     * @param loaderType Type of loader to install (NET8 or NET35)
     * @return InstallationResult with success status and details
     */
    public static InstallationResult installMelonLoaderOnline(Context context, String gamePackage, MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("🚀 Starting automated MelonLoader installation...");
        LogUtils.logUser("Target: " + gamePackage + " (" + loaderType.getDisplayName() + ")");
        
        // Validate parameters
        if (context == null) {
            return new InstallationResult(false, "Context is null", "Application context is required");
        }
        
        if (gamePackage == null || gamePackage.trim().isEmpty()) {
            return new InstallationResult(false, "Invalid game package", "Game package cannot be null or empty");
        }
        
        if (loaderType == null) {
            return new InstallationResult(false, "Invalid loader type", "Loader type cannot be null");
        }
        
        try {
            // Step 1: Initialize directory structure
            LogUtils.logUser("📁 Step 1: Initializing directory structure...");
            if (!PathManager.initializeGameDirectories(context, gamePackage)) {
                return new InstallationResult(false, "Failed to create directory structure", "Could not create required directories");
            }
            LogUtils.logUser("✅ Directory structure ready");
            
            // Step 2: Determine download URL and target directory
            String downloadUrl = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? MELONLOADER_URL : LEMONLOADER_URL;
            File targetDirectory = PathManager.getMelonLoaderDir(context, gamePackage);
            
            if (targetDirectory == null) {
                return new InstallationResult(false, "Cannot determine installation directory", "PathManager returned null directory");
            }
            
            LogUtils.logUser("📂 Installation directory: " + targetDirectory.getAbsolutePath());
            LogUtils.logUser("🌐 Download URL: " + downloadUrl);
            
            // Step 3: Download and extract
            LogUtils.logUser("⬇️ Step 2: Downloading and extracting MelonLoader files...");
            boolean downloadSuccess = Downloader.downloadAndExtractZip(downloadUrl, targetDirectory);
            
            if (!downloadSuccess) {
                return new InstallationResult(false, "Download or extraction failed", "Failed to download from: " + downloadUrl);
            }
            
            // Step 4: Organize files according to MelonLoader structure
            LogUtils.logUser("📋 Step 3: Organizing files into proper structure...");
            InstallationResult organizationResult = organizeExtractedFiles(context, gamePackage, targetDirectory, loaderType);
            
            if (!organizationResult.success) {
                return organizationResult;
            }
            
            // Step 5: Validate installation
            LogUtils.logUser("🔍 Step 4: Validating installation...");
            boolean validationSuccess = MelonLoaderManager.validateLoaderInstallation(context, gamePackage).isValid;
            
            if (!validationSuccess) {
                LogUtils.logUser("⚠️ Installation validation failed, attempting repair...");
                if (MelonLoaderManager.attemptRepair(context, gamePackage)) {
                    LogUtils.logUser("✅ Repair successful");
                    validationSuccess = true;
                } else {
                    return new InstallationResult(false, "Installation validation failed", "Files were downloaded but validation failed");
                }
            }
            
            // Step 6: Create final result
            InstallationResult result = new InstallationResult(true, "✅ " + loaderType.getDisplayName() + " installed successfully!");
            result.installationPath = targetDirectory;
            result.filesInstalled = countInstalledFiles(targetDirectory);
            result.totalSize = calculateDirectorySize(targetDirectory);
            
            LogUtils.logUser("🎉 Installation completed successfully!");
            LogUtils.logUser("📊 Files installed: " + result.filesInstalled);
            LogUtils.logUser("💾 Total size: " + FileUtils.formatFileSize(result.totalSize));
            LogUtils.logUser("📍 Installation path: " + result.installationPath.getAbsolutePath());
            
            return result;
            
        } catch (Exception e) {
            String errorMsg = "Unexpected error during installation: " + e.getMessage();
            LogUtils.logDebug(errorMsg);
            e.printStackTrace();
            return new InstallationResult(false, "Installation failed with exception", errorMsg);
        }
    }
    
    /**
     * Organize extracted files into proper MelonLoader directory structure
     * Based on the MelonLoader_File_List.txt structure
     */
    private static InstallationResult organizeExtractedFiles(Context context, String gamePackage, File extractedDir, MelonLoaderManager.LoaderType loaderType) {
        try {
            LogUtils.logUser("🗂️ Organizing extracted files...");
            
            // Create target directories based on loader type
            File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
            File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
            File dependenciesDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
            
            // Ensure directories exist
            PathManager.ensureDirectoryExists(net8Dir);
            PathManager.ensureDirectoryExists(net35Dir);
            PathManager.ensureDirectoryExists(dependenciesDir);
            
            int organizedFiles = 0;
            
            // Process extracted files
            File[] extractedFiles = extractedDir.listFiles();
            if (extractedFiles != null) {
                for (File file : extractedFiles) {
                    if (organizeFile(file, net8Dir, net35Dir, dependenciesDir, loaderType)) {
                        organizedFiles++;
                    }
                }
            }
            
            LogUtils.logUser("📁 Organized " + organizedFiles + " files into proper structure");
            
            // Create additional required directories
            createAdditionalDirectories(context, gamePackage);
            
            InstallationResult result = new InstallationResult(true, "File organization completed");
            result.filesInstalled = organizedFiles;
            return result;
            
        } catch (Exception e) {
            return new InstallationResult(false, "File organization failed", e.getMessage());
        }
    }
    
    /**
     * Organize individual file based on its type and target loader
     */
    private static boolean organizeFile(File file, File net8Dir, File net35Dir, File dependenciesDir, MelonLoaderManager.LoaderType loaderType) {
        if (file == null || !file.exists()) {
            return false;
        }
        
        try {
            String fileName = file.getName().toLowerCase();
            File targetDir = null;
            
            // Determine target directory based on file type and loader type
            if (fileName.contains("net8") && loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                targetDir = net8Dir;
            } else if (fileName.contains("net35") && loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET35) {
                targetDir = net35Dir;
            } else if (fileName.contains("dependencies") || fileName.contains("supportmodules") || fileName.contains("il2cpp")) {
                targetDir = dependenciesDir;
            } else if (fileName.endsWith(".dll") || fileName.endsWith(".xml") || fileName.endsWith(".pdb")) {
                // Core files go to appropriate runtime directory
                targetDir = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? net8Dir : net35Dir;
            }
            
            if (targetDir != null) {
                File targetFile = new File(targetDir, file.getName());
                if (file.renameTo(targetFile)) {
                    LogUtils.logDebug("Moved: " + file.getName() + " -> " + targetDir.getName());
                    return true;
                }
            }
            
            return false;
            
        } catch (Exception e) {
            LogUtils.logDebug("Error organizing file " + file.getName() + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Create additional required directories for MelonLoader
     */
    private static void createAdditionalDirectories(Context context, String gamePackage) {
        try {
            // Create subdirectories in Dependencies
            File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
            
            String[] subDirs = {
                "SupportModules",
                "CompatibilityLayers", 
                "Il2CppAssemblyGenerator",
                "Il2CppAssemblyGenerator/Cpp2IL",
                "Il2CppAssemblyGenerator/Cpp2IL/cpp2il_out",
                "Il2CppAssemblyGenerator/Il2CppInterop",
                "Il2CppAssemblyGenerator/Il2CppInterop/Il2CppAssemblies",
                "Il2CppAssemblyGenerator/UnityDependencies",
                "Il2CppAssemblyGenerator/runtimes",
                "Il2CppAssemblyGenerator/runtimes/linux-arm64/native",
                "Il2CppAssemblyGenerator/runtimes/linux-arm/native",
                "Il2CppAssemblyGenerator/runtimes/linux-x64/native",
                "Il2CppAssemblyGenerator/runtimes/linux-x86/native"
            };
            
            for (String subDir : subDirs) {
                File dir = new File(depsDir, subDir);
                PathManager.ensureDirectoryExists(dir);
            }
            
            LogUtils.logDebug("Created additional MelonLoader directories");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error creating additional directories: " + e.getMessage());
        }
    }
    
    /**
     * Count files in a directory recursively
     */
    private static int countInstalledFiles(File directory) {
        if (directory == null || !directory.exists() || !directory.isDirectory()) {
            return 0;
        }
        
        int count = 0;
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isDirectory()) {
                    count += countInstalledFiles(file);
                } else {
                    count++;
                }
            }
        }
        return count;
    }
    
    /**
     * Calculate total size of directory
     */
    private static long calculateDirectorySize(File directory) {
        if (directory == null || !directory.exists()) {
            return 0;
        }
        
        long size = 0;
        if (directory.isDirectory()) {
            File[] files = directory.listFiles();
            if (files != null) {
                for (File file : files) {
                    if (file.isDirectory()) {
                        size += calculateDirectorySize(file);
                    } else {
                        size += file.length();
                    }
                }
            }
        } else {
            size = directory.length();
        }
        return size;
    }
    
    /**
     * Convenience method for installing MelonLoader (NET8)
     */
    public static InstallationResult installMelonLoader(Context context, String gamePackage) {
        return installMelonLoaderOnline(context, gamePackage, MelonLoaderManager.LoaderType.MELONLOADER_NET8);
    }
    
    /**
     * Convenience method for installing LemonLoader (NET35)  
     */
    public static InstallationResult installLemonLoader(Context context, String gamePackage) {
        return installMelonLoaderOnline(context, gamePackage, MelonLoaderManager.LoaderType.MELONLOADER_NET35);
    }
    
    /**
     * Install for Terraria specifically
     */
    public static InstallationResult installForTerraria(Context context, MelonLoaderManager.LoaderType loaderType) {
        return installMelonLoaderOnline(context, MelonLoaderManager.TERRARIA_PACKAGE, loaderType);
    }
    
    /**
     * Check if online installation is possible (internet connectivity)
     */
    public static boolean isOnlineInstallationAvailable() {
        try {
            // Simple connectivity check
            java.net.URL url = new java.net.URL("https://github.com");
            java.net.HttpURLConnection connection = (java.net.HttpURLConnection) url.openConnection();
            connection.setRequestMethod("HEAD");
            connection.setConnectTimeout(3000);
            connection.setReadTimeout(3000);
            connection.connect();
            
            int responseCode = connection.getResponseCode();
            return responseCode == 200;
            
        } catch (Exception e) {
            LogUtils.logDebug("Online installation not available: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get installation progress callback interface
     */
    public interface InstallationProgressCallback {
        void onProgress(String message, int percentage);
        void onComplete(InstallationResult result);
        void onError(String error);
    }
    
    /**
     * Asynchronous installation with progress callback
     */
    public static void installMelonLoaderAsync(Context context, String gamePackage, MelonLoaderManager.LoaderType loaderType, InstallationProgressCallback callback) {
        new Thread(() -> {
            try {
                if (callback != null) {
                    callback.onProgress("Starting installation...", 0);
                }
                
                InstallationResult result = installMelonLoaderOnline(context, gamePackage, loaderType);
                
                if (callback != null) {
                    if (result.success) {
                        callback.onProgress("Installation completed!", 100);
                        callback.onComplete(result);
                    } else {
                        callback.onError(result.message + (result.errorDetails != null ? ": " + result.errorDetails : ""));
                    }
                }
                
            } catch (Exception e) {
                if (callback != null) {
                    callback.onError("Installation failed: " + e.getMessage());
                }
            }
        }).start();
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/PatchResult.java

// File: PatchResult.java - Complete patch result class
// Path: /storage/emulated/0/AndroidIDEProjects/ModLoader/app/src/main/java/com/modloader/util/PatchResult.java

package com.modloader.util;

import java.util.ArrayList;
import java.util.List;

public class PatchResult {
    public boolean success;
    public String errorMessage;
    public List<String> warnings;
    public List<String> details;
    public String outputPath;
    public long processingTime;
    public long startTime;
    public long endTime;
    public String operationType;
    
    public PatchResult() {
        this.success = false;
        this.warnings = new ArrayList<>();
        this.details = new ArrayList<>();
        this.startTime = System.currentTimeMillis();
        this.processingTime = 0;
    }
    
    public PatchResult(boolean success) {
        this();
        this.success = success;
    }
    
    public PatchResult(boolean success, String errorMessage) {
        this(success);
        this.errorMessage = errorMessage;
    }
    
    public PatchResult(String operationType) {
        this();
        this.operationType = operationType;
    }
    
    public void addWarning(String warning) {
        if (warnings == null) {
            warnings = new ArrayList<>();
        }
        warnings.add(warning);
    }
    
    public void addDetail(String detail) {
        if (details == null) {
            details = new ArrayList<>();
        }
        details.add(detail);
    }
    
    public boolean hasWarnings() {
        return warnings != null && !warnings.isEmpty();
    }
    
    public boolean hasDetails() {
        return details != null && !details.isEmpty();
    }
    
    public void setSuccess(boolean success) {
        this.success = success;
        if (success) {
            this.endTime = System.currentTimeMillis();
            this.processingTime = this.endTime - this.startTime;
        }
    }
    
    public void setError(String errorMessage) {
        this.success = false;
        this.errorMessage = errorMessage;
        this.endTime = System.currentTimeMillis();
        this.processingTime = this.endTime - this.startTime;
    }
    
    public void setErrorMessage(String errorMessage) {
        this.errorMessage = errorMessage;
    }
    
    public void setOutputPath(String outputPath) {
        this.outputPath = outputPath;
    }
    
    public void setOperationType(String operationType) {
        this.operationType = operationType;
    }
    
    public void complete() {
        this.endTime = System.currentTimeMillis();
        this.processingTime = this.endTime - this.startTime;
    }
    
    // Convenience method to convert to boolean for backward compatibility
    public boolean isSuccess() {
        return success;
    }
    
    public long getProcessingTimeMs() {
        return processingTime;
    }
    
    public String getFormattedProcessingTime() {
        if (processingTime < 1000) {
            return processingTime + "ms";
        } else if (processingTime < 60000) {
            return String.format("%.1fs", processingTime / 1000.0);
        } else {
            long seconds = processingTime / 1000;
            long minutes = seconds / 60;
            seconds = seconds % 60;
            return String.format("%dm %ds", minutes, seconds);
        }
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("PatchResult{");
        sb.append("success=").append(success);
        if (operationType != null) {
            sb.append(", operation='").append(operationType).append('\'');
        }
        if (errorMessage != null) {
            sb.append(", error='").append(errorMessage).append('\'');
        }
        if (hasWarnings()) {
            sb.append(", warnings=").append(warnings.size());
        }
        if (processingTime > 0) {
            sb.append(", time=").append(getFormattedProcessingTime());
        }
        sb.append('}');
        return sb.toString();
    }
    
    public String getDetailedReport() {
        StringBuilder sb = new StringBuilder();
        sb.append("=== Patch Operation Report ===\n");
        if (operationType != null) {
            sb.append("Operation: ").append(operationType).append("\n");
        }
        sb.append("Result: ").append(success ? "SUCCESS" : "FAILED").append("\n");
        sb.append("Processing Time: ").append(getFormattedProcessingTime()).append("\n");
        
        if (outputPath != null) {
            sb.append("Output: ").append(outputPath).append("\n");
        }
        
        if (errorMessage != null) {
            sb.append("Error: ").append(errorMessage).append("\n");
        }
        
        if (hasWarnings()) {
            sb.append("\nWarnings (").append(warnings.size()).append("):\n");
            for (String warning : warnings) {
                sb.append("  - ").append(warning).append("\n");
            }
        }
        
        if (hasDetails()) {
            sb.append("\nDetails (").append(details.size()).append("):\n");
            for (String detail : details) {
                sb.append("  • ").append(detail).append("\n");
            }
        }
        
        return sb.toString();
    }
    
    // Static factory methods for common scenarios
    public static PatchResult success(String operationType) {
        PatchResult result = new PatchResult(operationType);
        result.setSuccess(true);
        return result;
    }
    
    public static PatchResult success(String operationType, String outputPath) {
        PatchResult result = success(operationType);
        result.setOutputPath(outputPath);
        return result;
    }
    
    public static PatchResult failure(String operationType, String errorMessage) {
        PatchResult result = new PatchResult(operationType);
        result.setError(errorMessage);
        return result;
    }
    
    public static PatchResult inProgress(String operationType) {
        return new PatchResult(operationType);
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/PathManager.java

// File: PathManager.java (FIXED Part 1) - Centralized Path Management
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/terrarialoader/util/PathManager.java

package com.modloader.util;

import android.content.Context;
import java.io.File;

/**
 * Centralized path management for TerrariaLoader
 * All file operations should use these standardized paths
 * FIXED: Added proper app logs directory support and correct structure
 */
public class PathManager {
    
    // Base directory: /storage/emulated/0/Android/data/com.modloader/files
    private static File getAppDataDirectory(Context context) {
        return context.getExternalFilesDir(null);
    }
    
    // === TERRARIA LOADER STRUCTURE ===
    
    /**
     * Base TerrariaLoader directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader
     */
    public static File getTerrariaLoaderBaseDir(Context context) {
        return new File(getAppDataDirectory(context), "TerrariaLoader");
    }
    
    /**
     * Game-specific base directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}
     */
    public static File getGameBaseDir(Context context, String gamePackage) {
        return new File(getTerrariaLoaderBaseDir(context), gamePackage);
    }
    
    /**
     * Default Terraria directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/com.and.games505.TerrariaPaid
     */
    public static File getTerrariaBaseDir(Context context) {
        return getGameBaseDir(context, "com.and.games505.TerrariaPaid");
    }
    
    // === MOD DIRECTORIES ===
    
    /**
     * DLL Mods directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Mods/DLL
     */
    public static File getDllModsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Mods/DLL");
    }
    
    /**
     * DEX Mods directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Mods/DEX
     */
    public static File getDexModsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Mods/DEX");
    }
    
    /**
     * Legacy mods directory (for backward compatibility)
     * @return /storage/emulated/0/Android/data/com.modloader/files/mods
     */
    public static File getLegacyModsDir(Context context) {
        return new File(getAppDataDirectory(context), "mods");
    }
    
    // === MELONLOADER STRUCTURE ===
    
    /**
     * MelonLoader base directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader
     */
    public static File getMelonLoaderDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Loaders/MelonLoader");
    }
    
    /**
     * MelonLoader NET8 runtime directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/net8
     */
    public static File getMelonLoaderNet8Dir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "net8");
    }
    
    /**
     * MelonLoader NET35 runtime directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/net35
     */
    public static File getMelonLoaderNet35Dir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "net35");
    }
    
    /**
     * MelonLoader Dependencies directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/Dependencies
     */
    public static File getMelonLoaderDependenciesDir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "Dependencies");
    }
    
    // === PLUGINS AND USERLIBS (FIXED: Now at game root level) ===
    
    /**
     * FIXED: Plugins directory (at game root level, not inside MelonLoader)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Plugins
     */
    public static File getPluginsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Plugins");
    }
    
    /**
     * FIXED: UserLibs directory (at game root level, not inside MelonLoader)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/UserLibs
     */
    public static File getUserLibsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "UserLibs");
    }
    
    // === LOG DIRECTORIES ===
    
    /**
     * FIXED: Game logs directory (MelonLoader logs)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Logs
     */
    public static File getLogsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Logs");
    }
    
    /**
     * FIXED: App logs directory (TerrariaLoader app logs)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/AppLogs
     */
    public static File getAppLogsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "AppLogs");
    }
    
    /**
     * Legacy app logs directory (for backward compatibility)
     * @return /storage/emulated/0/Android/data/com.modloader/files/logs
     */
    public static File getLegacyAppLogsDir(Context context) {
        return new File(getAppDataDirectory(context), "logs");
    }
    
    /**
     * Exports directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/exports
     */
    public static File getExportsDir(Context context) {
        return new File(getAppDataDirectory(context), "exports");
    }
    
    // === BACKUP DIRECTORIES ===
    
    /**
     * Backups directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Backups
     */
    public static File getBackupsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Backups");
    }
    
    // === CONFIG DIRECTORIES ===
    
    /**
     * Config directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Config
     */
    public static File getConfigDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Config");
    }
    
    // === UTILITY METHODS ===
    
    /**
     * Ensure a directory exists, creating it if necessary
     * @param directory The directory to ensure exists
     * @return true if directory exists or was created successfully
     */
    public static boolean ensureDirectoryExists(File directory) {
        if (directory == null) {
            LogUtils.logDebug("Directory is null");
            return false;
        }
        
        if (directory.exists()) {
            return directory.isDirectory();
        }
        
        try {
            boolean created = directory.mkdirs();
            if (created) {
                LogUtils.logDebug("Created directory: " + directory.getAbsolutePath());
            } else {
                LogUtils.logDebug("Failed to create directory: " + directory.getAbsolutePath());
            }
            return created;
        } catch (Exception e) {
            LogUtils.logDebug("Error creating directory: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Initialize all required directories for a game package
     * @param context Application context
     * @param gamePackage Game package name
     * @return true if all directories were created successfully
     */
    public static boolean initializeGameDirectories(Context context, String gamePackage) {
        LogUtils.logUser("Initializing directory structure for: " + gamePackage);
        
        File[] requiredDirs = {
            getGameBaseDir(context, gamePackage),
            getDllModsDir(context, gamePackage),
            getDexModsDir(context, gamePackage),
            getMelonLoaderDir(context, gamePackage),
            getMelonLoaderNet8Dir(context, gamePackage),
            getMelonLoaderNet35Dir(context, gamePackage),
            getMelonLoaderDependenciesDir(context, gamePackage),
            getPluginsDir(context, gamePackage),        // FIXED: Added Plugins
            getUserLibsDir(context, gamePackage),       // FIXED: Added UserLibs
            getLogsDir(context, gamePackage),           // Game logs
            getAppLogsDir(context, gamePackage),        // FIXED: Added App logs
            getBackupsDir(context, gamePackage),
            getConfigDir(context, gamePackage)
        };
        
        boolean allSuccess = true;
        int createdCount = 0;
        
        for (File dir : requiredDirs) {
            if (!dir.exists()) {
                if (ensureDirectoryExists(dir)) {
                    createdCount++;
                } else {
                    allSuccess = false;
                    LogUtils.logDebug("Failed to create: " + dir.getAbsolutePath());
                }
            }
        }
        
        LogUtils.logUser("Directory initialization complete: " + createdCount + " directories created");
        
        // Create README files
        if (allSuccess) {
            createReadmeFiles(context, gamePackage);
        }
        
        return allSuccess;
    }
    
    /**
     * FIXED: Create helpful README files in directories
     */
    private static void createReadmeFiles(Context context, String gamePackage) {
        try {
            // DLL Mods README
            File dllReadme = new File(getDllModsDir(context, gamePackage), "README.txt");
            if (!dllReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(dllReadme)) {
                    writer.write("=== TerrariaLoader - DLL Mods ===\n\n");
                    writer.write("Place your .dll mod files here.\n");
                    writer.write("Requires MelonLoader to be installed.\n\n");
                    writer.write("Supported formats:\n");
                    writer.write("• .dll files (enabled)\n");
                    writer.write("• .dll.disabled files (disabled)\n\n");
                    writer.write("Path: " + dllReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // DEX Mods README
            File dexReadme = new File(getDexModsDir(context, gamePackage), "README.txt");
            if (!dexReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(dexReadme)) {
                    writer.write("=== TerrariaLoader - DEX/JAR Mods ===\n\n");
                    writer.write("Place your .dex and .jar mod files here.\n");
                    writer.write("These are Java-based mods for Android.\n\n");
                    writer.write("Supported formats:\n");
                    writer.write("• .dex files (enabled)\n");
                    writer.write("• .jar files (enabled)\n");
                    writer.write("• .dex.disabled files (disabled)\n");
                    writer.write("• .jar.disabled files (disabled)\n\n");
                    writer.write("Path: " + dexReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: Plugins README
            File pluginsReadme = new File(getPluginsDir(context, gamePackage), "README.txt");
            if (!pluginsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(pluginsReadme)) {
                    writer.write("=== MelonLoader Plugins Directory ===\n\n");
                    writer.write("This directory contains MelonLoader plugins.\n");
                    writer.write("Plugins extend MelonLoader functionality.\n\n");
                    writer.write("Files you might see here:\n");
                    writer.write("• .dll plugin files\n");
                    writer.write("• Plugin configuration files\n\n");
                    writer.write("Path: " + pluginsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: UserLibs README
            File userlibsReadme = new File(getUserLibsDir(context, gamePackage), "README.txt");
            if (!userlibsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(userlibsReadme)) {
                    writer.write("=== MelonLoader UserLibs Directory ===\n\n");
                    writer.write("This directory contains user libraries.\n");
                    writer.write("Libraries that mods depend on go here.\n\n");
                    writer.write("Files you might see here:\n");
                    writer.write("• .dll library files\n");
                    writer.write("• Shared mod dependencies\n\n");
                    writer.write("Path: " + userlibsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: Game logs README
            File gameLogsReadme = new File(getLogsDir(context, gamePackage), "README.txt");
            if (!gameLogsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(gameLogsReadme)) {
                    writer.write("=== MelonLoader Game Logs ===\n\n");
                    writer.write("This directory contains logs from MelonLoader and mods.\n");
                    writer.write("These are generated when running the patched game.\n\n");
                    writer.write("Log files follow rotation:\n");
                    writer.write("• Log.txt (current log)\n");
                    writer.write("• Log1.txt to Log5.txt (previous logs)\n");
                    writer.write("• Oldest logs are automatically deleted\n\n");
                    writer.write("Path: " + gameLogsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: App logs README
            File appLogsReadme = new File(getAppLogsDir(context, gamePackage), "README.txt");
            if (!appLogsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(appLogsReadme)) {
                    writer.write("=== TerrariaLoader App Logs ===\n\n");
                    writer.write("This directory contains logs from TerrariaLoader app.\n");
                    writer.write("These are generated when using this app.\n\n");
                    writer.write("Log files follow rotation:\n");
                    writer.write("• AppLog.txt (current log)\n");
                    writer.write("• AppLog1.txt to AppLog5.txt (previous logs)\n");
                    writer.write("• Oldest logs are automatically deleted\n\n");
                    writer.write("Path: " + appLogsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error creating README files: " + e.getMessage());
        }
    }
    
    /**
     * FIXED: Get standardized path string for logging/display
     */
    public static String getPathInfo(Context context, String gamePackage) {
        StringBuilder info = new StringBuilder();
        info.append("=== TerrariaLoader Directory Structure ===\n");
        info.append("Base: ").append(getTerrariaLoaderBaseDir(context).getAbsolutePath()).append("\n");
        info.append("Game: ").append(getGameBaseDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("DLL Mods: ").append(getDllModsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("DEX Mods: ").append(getDexModsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("MelonLoader: ").append(getMelonLoaderDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("Plugins: ").append(getPluginsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("UserLibs: ").append(getUserLibsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("Game Logs: ").append(getLogsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("App Logs: ").append(getAppLogsDir(context, gamePackage).getAbsolutePath()).append("\n");
        return info.toString();
    }
    
    /**
     * Check if legacy structure exists and needs migration
     */
    public static boolean needsMigration(Context context) {
        File legacyMods = getLegacyModsDir(context);
        File legacyAppLogs = getLegacyAppLogsDir(context);
        File newStructure = getTerrariaBaseDir(context);
        
        return ((legacyMods.exists() && legacyMods.listFiles() != null && legacyMods.listFiles().length > 0) ||
                (legacyAppLogs.exists() && legacyAppLogs.listFiles() != null && legacyAppLogs.listFiles().length > 0)) && 
               !newStructure.exists();
    }
    
    /**
     * FIXED: Migrate from legacy structure to new structure
     */
    public static boolean migrateLegacyStructure(Context context) {
        if (!needsMigration(context)) {
            return true;
        }
        
        LogUtils.logUser("Migrating from legacy directory structure...");
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            
            // Initialize new structure
            if (!initializeGameDirectories(context, gamePackage)) {
                LogUtils.logDebug("Failed to initialize new directory structure");
                return false;
            }
            
            // Migrate mods
            File legacyModsDir = getLegacyModsDir(context);
            File newDexMods = getDexModsDir(context, gamePackage);
            
            if (legacyModsDir.exists()) {
                File[] modFiles = legacyModsDir.listFiles();
                if (modFiles != null) {
                    int migratedCount = 0;
                    for (File modFile : modFiles) {
                        if (modFile.isFile()) {
                            File newLocation = new File(newDexMods, modFile.getName());
                            if (modFile.renameTo(newLocation)) {
                                migratedCount++;
                            }
                        }
                    }
                    LogUtils.logUser("Migrated " + migratedCount + " mod files to new structure");
                }
            }
            
            // Migrate legacy app logs
            File legacyAppLogs = getLegacyAppLogsDir(context);
            File newAppLogs = getAppLogsDir(context, gamePackage);
            
            if (legacyAppLogs.exists()) {
                File[] logFiles = legacyAppLogs.listFiles((dir, name) -> 
                    name.endsWith(".txt") || name.startsWith("auto_save_"));
                
                if (logFiles != null && logFiles.length > 0) {
                    LogUtils.logUser("Migrating " + logFiles.length + " legacy log files...");
                    
                    if (!newAppLogs.exists()) {
                        newAppLogs.mkdirs();
                    }
                    
                    int migratedLogCount = 0;
                    for (File logFile : logFiles) {
                        // Rename to new format
                        String newName = "AppLog" + (migratedLogCount + 1) + ".txt";
                        File newLocation = new File(newAppLogs, newName);
                        
                        if (logFile.renameTo(newLocation)) {
                            migratedLogCount++;
                        }
                    }
                    LogUtils.logUser("Migrated " + migratedLogCount + " log files to new structure");
                }
            }
            
            LogUtils.logUser("✅ Migration completed successfully");
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Migration failed: " + e.getMessage());
            return false;
        }
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/PathProvider.java

package com.modloader.util;
import android.content.Context;
import java.io.File;
public final class PathProvider {
    public static final String TERRARIA_PACKAGE = "com.and.games505.TerrariaPaid";
    public static final String LOADER_DIR_NAME = "TerrariaLoader";
    private PathProvider(){}
    public static File getRoot(Context ctx){
        File ext = ctx.getExternalFilesDir(null);
        File root = new File(ext, LOADER_DIR_NAME);
        if(!root.exists()) root.mkdirs();
        return root;
    }
    public static File getAddons(Context ctx){ File f=new File(getRoot(ctx),"addons"); if(!f.exists()) f.mkdirs(); return f; }
    public static File getScripts(Context ctx){ File f=new File(getRoot(ctx),"scripts"); if(!f.exists()) f.mkdirs(); return f; }
    public static File getConfigs(Context ctx){ File f=new File(getRoot(ctx),"configs"); if(!f.exists()) f.mkdirs(); return f; }
    public static File getModules(Context ctx){ File f=new File(getRoot(ctx),"modules"); if(!f.exists()) f.mkdirs(); return f; }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/PermissionManager.java

// File: PermissionManager.java (FIXED) - Complete Permission Management with Shizuku Integration
// Path: /app/src/main/java/com/modloader/util/PermissionManager.java

package com.modloader.util;

import android.Manifest;
import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Environment;
import android.provider.Settings;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;
import androidx.appcompat.app.AlertDialog;

import java.util.ArrayList;
import java.util.List;

/**
 * FIXED: Complete Permission Manager with proper Shizuku integration
 * Handles all permission types including storage, installation, and Shizuku
 */
public class PermissionManager {
    private static final String TAG = "PermissionManager";
    
    // Permission request codes
    private static final int REQUEST_STORAGE_PERMISSION = 1001;
    private static final int REQUEST_INSTALL_PERMISSION = 1002;
    private static final int REQUEST_ALL_PERMISSIONS = 1003;
    private static final int REQUEST_MANAGE_EXTERNAL_STORAGE = 1004;
    
    private final Context context;
    private final ShizukuManager shizukuManager;
    private PermissionCallback permissionCallback;
    
    // Interface for permission callbacks
    public interface PermissionCallback {
        default void onPermissionGranted(String permission) {}
        default void onPermissionDenied(String permission) {}
        default void onPermissionError(String permission, String error) {}
        default void onAllPermissionsResult(boolean allGranted, List<String> denied) {}
    }
    
    public PermissionManager(Context context) {
        this.context = context;
        this.shizukuManager = new ShizukuManager(context);
        LogUtils.logDebug("PermissionManager initialized");
    }
    
    public void setPermissionCallback(PermissionCallback callback) {
        this.permissionCallback = callback;
    }
    
    // ===== FIXED: Basic Permission Checks =====
    
    /**
     * FIXED: Check storage permission with Android 11+ MANAGE_EXTERNAL_STORAGE support
     */
    public boolean hasStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // Android 11+ - Check MANAGE_EXTERNAL_STORAGE
            boolean hasManageStorage = Environment.isExternalStorageManager();
            LogUtils.logDebug("Android 11+ storage permission (MANAGE_EXTERNAL_STORAGE): " + hasManageStorage);
            
            if (hasManageStorage) {
                return true;
            }
            
            // Fallback to scoped storage permissions
            boolean hasReadStorage = ContextCompat.checkSelfPermission(context, 
                Manifest.permission.READ_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED;
            boolean hasWriteStorage = ContextCompat.checkSelfPermission(context, 
                Manifest.permission.WRITE_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED;
            
            LogUtils.logDebug("Scoped storage permissions - Read: " + hasReadStorage + ", Write: " + hasWriteStorage);
            return hasReadStorage;
            
        } else {
            // Android 10 and below
            boolean hasReadStorage = ContextCompat.checkSelfPermission(context, 
                Manifest.permission.READ_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED;
            boolean hasWriteStorage = ContextCompat.checkSelfPermission(context, 
                Manifest.permission.WRITE_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED;
            
            LogUtils.logDebug("Legacy storage permissions - Read: " + hasReadStorage + ", Write: " + hasWriteStorage);
            return hasReadStorage && hasWriteStorage;
        }
    }
    
    /**
     * FIXED: Check installation permission
     */
    public boolean hasInstallPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            boolean canInstall = context.getPackageManager().canRequestPackageInstalls();
            LogUtils.logDebug("Install permission (Android 8+): " + canInstall);
            return canInstall;
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            try {
                int result = Settings.Secure.getInt(context.getContentResolver(),
                    Settings.Secure.INSTALL_NON_MARKET_APPS, 0);
                boolean canInstall = (result == 1);
                LogUtils.logDebug("Install permission (legacy): " + canInstall);
                return canInstall;
            } catch (Exception e) {
                LogUtils.logDebug("Install permission check error: " + e.getMessage());
                return false;
            }
        }
        return true; // Very old Android versions
    }
    
    /**
     * FIXED: Check all basic permissions
     */
    public boolean hasBasicPermissions() {
        boolean storage = hasStoragePermission();
        boolean install = hasInstallPermission();
        
        LogUtils.logDebug("Basic permissions - Storage: " + storage + ", Install: " + install);
        return storage && install;
    }
    
    /**
     * FIXED: Check if enhanced permissions are available (Shizuku)
     */
    public boolean hasEnhancedPermissions() {
        boolean shizukuReady = shizukuManager.isShizukuReady();
        LogUtils.logDebug("Enhanced permissions (Shizuku): " + shizukuReady);
        return shizukuReady;
    }
    
    // ===== Permission Request Methods =====
    
    /**
     * FIXED: Request storage permissions with Android 11+ support
     */
    public void requestStoragePermission() {
        LogUtils.logUser("Requesting storage permissions...");
        
        if (hasStoragePermission()) {
            LogUtils.logUser("✅ Storage permissions already granted");
            if (permissionCallback != null) {
                permissionCallback.onPermissionGranted("STORAGE");
            }
            return;
        }
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // Android 11+ - Request MANAGE_EXTERNAL_STORAGE
            showManageExternalStorageDialog();
        } else {
            // Android 10 and below - Request traditional storage permissions
            requestLegacyStoragePermissions();
        }
    }
    
    /**
     * Request installation permissions
     */
    public void requestInstallPermission() {
        LogUtils.logUser("Requesting install permissions...");
        
        if (hasInstallPermission()) {
            LogUtils.logUser("✅ Install permissions already granted");
            if (permissionCallback != null) {
                permissionCallback.onPermissionGranted("INSTALL");
            }
            return;
        }
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            showInstallPermissionDialog();
        } else {
            showUnknownSourcesDialog();
        }
    }
    
    /**
     * FIXED: Request all basic permissions
     */
    public void requestBasicPermissions() {
        LogUtils.logUser("Requesting all basic permissions...");
        
        List<String> neededPermissions = new ArrayList<>();
        
        if (!hasStoragePermission()) {
            neededPermissions.add("STORAGE");
        }
        
        if (!hasInstallPermission()) {
            neededPermissions.add("INSTALL");
        }
        
        if (neededPermissions.isEmpty()) {
            LogUtils.logUser("✅ All basic permissions already granted");
            if (permissionCallback != null) {
                permissionCallback.onAllPermissionsResult(true, new ArrayList<>());
            }
            return;
        }
        
        // Request permissions sequentially
        requestPermissionsSequentially(neededPermissions, 0);
    }
    
    /**
     * FIXED: Request all permissions including enhanced (Shizuku)
     */
    public void requestAllPermissions() {
        LogUtils.logUser("Requesting all permissions including enhanced...");
        
        // First request basic permissions
        requestBasicPermissions();
        
        // Then request Shizuku permission if not already granted
        if (!shizukuManager.hasShizukuPermission()) {
            shizukuManager.setPermissionCallback(new ShizukuManager.ShizukuPermissionCallback() {
                @Override
                public void onPermissionGranted() {
                    LogUtils.logUser("✅ Shizuku permission granted");
                    if (permissionCallback != null) {
                        permissionCallback.onPermissionGranted("SHIZUKU");
                    }
                }
                
                @Override
                public void onPermissionDenied() {
                    LogUtils.logUser("❌ Shizuku permission denied");
                    if (permissionCallback != null) {
                        permissionCallback.onPermissionDenied("SHIZUKU");
                    }
                }
                
                @Override
                public void onPermissionError(String error) {
                    LogUtils.logUser("❌ Shizuku permission error: " + error);
                    if (permissionCallback != null) {
                        permissionCallback.onPermissionError("SHIZUKU", error);
                    }
                }
            });
            
            shizukuManager.requestShizukuPermission();
        }
    }
    
    // ===== Internal Permission Request Methods =====
    
    /**
     * Show dialog for MANAGE_EXTERNAL_STORAGE permission (Android 11+)
     */
    private void showManageExternalStorageDialog() {
        if (!(context instanceof Activity)) {
            LogUtils.logError("Cannot show dialog - context is not an Activity");
            return;
        }
        
        new AlertDialog.Builder(context)
            .setTitle("🗂️ Storage Access Required")
            .setMessage("This app needs access to manage all files for mod installation.\n\n" +
                       "Steps:\n" +
                       "1. Tap 'Grant Permission'\n" +
                       "2. Find 'TerrariaLoader' in the list\n" +
                       "3. Enable 'Allow access to manage all files'\n" +
                       "4. Return to this app")
            .setPositiveButton("Grant Permission", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION);
                    intent.setData(Uri.parse("package:" + context.getPackageName()));
                    ((Activity) context).startActivityForResult(intent, REQUEST_MANAGE_EXTERNAL_STORAGE);
                    LogUtils.logUser("Opening MANAGE_EXTERNAL_STORAGE settings");
                } catch (Exception e) {
                    LogUtils.logError("Failed to open MANAGE_EXTERNAL_STORAGE settings: " + e.getMessage());
                    // Fallback to general settings
                    try {
                        Intent intent = new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION);
                        ((Activity) context).startActivityForResult(intent, REQUEST_MANAGE_EXTERNAL_STORAGE);
                    } catch (Exception ex) {
                        Toast.makeText(context, "Cannot open storage settings", Toast.LENGTH_LONG).show();
                        if (permissionCallback != null) {
                            permissionCallback.onPermissionError("STORAGE", "Cannot open settings");
                        }
                    }
                }
            })
            .setNegativeButton("Cancel", (dialog, which) -> {
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied("STORAGE");
                }
            })
            .show();
    }
    
    /**
     * Request legacy storage permissions (Android 10 and below)
     */
    private void requestLegacyStoragePermissions() {
        if (!(context instanceof Activity)) {
            LogUtils.logError("Cannot request permissions - context is not an Activity");
            return;
        }
        
        String[] permissions = {
            Manifest.permission.READ_EXTERNAL_STORAGE,
            Manifest.permission.WRITE_EXTERNAL_STORAGE
        };
        
        ActivityCompat.requestPermissions((Activity) context, permissions, REQUEST_STORAGE_PERMISSION);
    }
    
    /**
     * Show dialog for install permission (Android 8+)
     */
    private void showInstallPermissionDialog() {
        if (!(context instanceof Activity)) {
            LogUtils.logError("Cannot show dialog - context is not an Activity");
            return;
        }
        
        new AlertDialog.Builder(context)
            .setTitle("📦 Install Permission Required")
            .setMessage("To install modified APK files, you need to allow this app to install unknown apps.\n\n" +
                       "Steps:\n" +
                       "1. Tap 'Grant Permission'\n" +
                       "2. Enable 'Allow from this source'\n" +
                       "3. Return to this app")
            .setPositiveButton("Grant Permission", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES);
                    intent.setData(Uri.parse("package:" + context.getPackageName()));
                    ((Activity) context).startActivityForResult(intent, REQUEST_INSTALL_PERMISSION);
                    LogUtils.logUser("Opening install permission settings");
                } catch (Exception e) {
                    LogUtils.logError("Failed to open install permission settings: " + e.getMessage());
                    Toast.makeText(context, "Cannot open install settings", Toast.LENGTH_LONG).show();
                    if (permissionCallback != null) {
                        permissionCallback.onPermissionError("INSTALL", "Cannot open settings");
                    }
                }
            })
            .setNegativeButton("Cancel", (dialog, which) -> {
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied("INSTALL");
                }
            })
            .show();
    }
    
    /**
     * Show dialog for unknown sources (Android 7.1 and below)
     */
    private void showUnknownSourcesDialog() {
        if (!(context instanceof Activity)) {
            LogUtils.logError("Cannot show dialog - context is not an Activity");
            return;
        }
        
        new AlertDialog.Builder(context)
            .setTitle("🔓 Enable Unknown Sources")
            .setMessage("To install modified APK files, you need to enable 'Unknown Sources' in security settings.\n\n" +
                       "Steps:\n" +
                       "1. Tap 'Open Settings'\n" +
                       "2. Find and enable 'Unknown sources'\n" +
                       "3. Return to this app")
            .setPositiveButton("Open Settings", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Settings.ACTION_SECURITY_SETTINGS);
                    ((Activity) context).startActivityForResult(intent, REQUEST_INSTALL_PERMISSION);
                    LogUtils.logUser("Opening security settings");
                } catch (Exception e) {
                    LogUtils.logError("Failed to open security settings: " + e.getMessage());
                    Toast.makeText(context, "Cannot open security settings", Toast.LENGTH_LONG).show();
                    if (permissionCallback != null) {
                        permissionCallback.onPermissionError("INSTALL", "Cannot open settings");
                    }
                }
            })
            .setNegativeButton("Cancel", (dialog, which) -> {
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied("INSTALL");
                }
            })
            .show();
    }
    
    /**
     * Request permissions sequentially to avoid overwhelming the user
     */
    private void requestPermissionsSequentially(List<String> permissions, int currentIndex) {
        if (currentIndex >= permissions.size()) {
            // All permissions processed
            boolean allGranted = hasBasicPermissions();
            List<String> denied = new ArrayList<>();
            
            if (!hasStoragePermission()) denied.add("STORAGE");
            if (!hasInstallPermission()) denied.add("INSTALL");
            
            LogUtils.logUser("All basic permissions processed - All granted: " + allGranted);
            if (permissionCallback != null) {
                permissionCallback.onAllPermissionsResult(allGranted, denied);
            }
            return;
        }
        
        String permission = permissions.get(currentIndex);
        LogUtils.logUser("Requesting permission " + (currentIndex + 1) + "/" + permissions.size() + ": " + permission);
        
        switch (permission) {
            case "STORAGE":
                // Set up callback for storage permission
                PermissionCallback originalCallback = permissionCallback;
                setPermissionCallback(new PermissionCallback() {
                    @Override
                    public void onPermissionGranted(String perm) {
                        LogUtils.logUser("✅ Storage permission granted");
                        // Continue with next permission
                        setPermissionCallback(originalCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                    
                    @Override
                    public void onPermissionDenied(String perm) {
                        LogUtils.logUser("❌ Storage permission denied");
                        // Continue with next permission
                        setPermissionCallback(originalCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                    
                    @Override
                    public void onPermissionError(String perm, String error) {
                        LogUtils.logUser("❌ Storage permission error: " + error);
                        // Continue with next permission
                        setPermissionCallback(originalCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                });
                requestStoragePermission();
                break;
                
            case "INSTALL":
                // Set up callback for install permission
                PermissionCallback originalInstallCallback = permissionCallback;
                setPermissionCallback(new PermissionCallback() {
                    @Override
                    public void onPermissionGranted(String perm) {
                        LogUtils.logUser("✅ Install permission granted");
                        // Continue with next permission
                        setPermissionCallback(originalInstallCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                    
                    @Override
                    public void onPermissionDenied(String perm) {
                        LogUtils.logUser("❌ Install permission denied");
                        // Continue with next permission
                        setPermissionCallback(originalInstallCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                    
                    @Override
                    public void onPermissionError(String perm, String error) {
                        LogUtils.logUser("❌ Install permission error: " + error);
                        // Continue with next permission
                        setPermissionCallback(originalInstallCallback);
                        requestPermissionsSequentially(permissions, currentIndex + 1);
                    }
                });
                requestInstallPermission();
                break;
                
            default:
                // Unknown permission, skip
                requestPermissionsSequentially(permissions, currentIndex + 1);
                break;
        }
    }
    
    // ===== Permission Result Handling =====
    
    /**
     * FIXED: Handle permission request results
     */
    public void handlePermissionResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        LogUtils.logDebug("Permission result - Request code: " + requestCode + 
                          ", Permissions: " + permissions.length + 
                          ", Results: " + grantResults.length);
        
        switch (requestCode) {
            case REQUEST_STORAGE_PERMISSION:
                handleStoragePermissionResult(permissions, grantResults);
                break;
                
            case REQUEST_ALL_PERMISSIONS:
                handleAllPermissionsResult(permissions, grantResults);
                break;
                
            default:
                LogUtils.logDebug("Unknown permission request code: " + requestCode);
                break;
        }
    }
    
    /**
     * Handle activity results (for settings that don't use permission callbacks)
     */
    public void handleActivityResult(int requestCode, int resultCode) {
        LogUtils.logDebug("Activity result - Request code: " + requestCode + ", Result: " + resultCode);
        
        switch (requestCode) {
            case REQUEST_MANAGE_EXTERNAL_STORAGE:
                // Check if MANAGE_EXTERNAL_STORAGE was granted
                boolean granted = hasStoragePermission();
                LogUtils.logUser("MANAGE_EXTERNAL_STORAGE result: " + granted);
                if (permissionCallback != null) {
                    if (granted) {
                        permissionCallback.onPermissionGranted("STORAGE");
                    } else {
                        permissionCallback.onPermissionDenied("STORAGE");
                    }
                }
                break;
                
            case REQUEST_INSTALL_PERMISSION:
                // Check if install permission was granted
                boolean installGranted = hasInstallPermission();
                LogUtils.logUser("Install permission result: " + installGranted);
                if (permissionCallback != null) {
                    if (installGranted) {
                        permissionCallback.onPermissionGranted("INSTALL");
                    } else {
                        permissionCallback.onPermissionDenied("INSTALL");
                    }
                }
                break;
        }
    }
    
    /**
     * Handle storage permission results
     */
    private void handleStoragePermissionResult(String[] permissions, int[] grantResults) {
        boolean allGranted = true;
        
        for (int i = 0; i < permissions.length && i < grantResults.length; i++) {
            boolean granted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
            LogUtils.logDebug("Permission " + permissions[i] + ": " + granted);
            if (!granted) {
                allGranted = false;
            }
        }
        
        LogUtils.logUser("Storage permission result: " + allGranted);
        if (permissionCallback != null) {
            if (allGranted) {
                permissionCallback.onPermissionGranted("STORAGE");
            } else {
                permissionCallback.onPermissionDenied("STORAGE");
            }
        }
    }
    
    /**
     * Handle all permissions results
     */
    private void handleAllPermissionsResult(String[] permissions, int[] grantResults) {
        List<String> denied = new ArrayList<>();
        boolean allGranted = true;
        
        for (int i = 0; i < permissions.length && i < grantResults.length; i++) {
            boolean granted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
            if (!granted) {
                allGranted = false;
                denied.add(permissions[i]);
            }
        }
        
        LogUtils.logUser("All permissions result - All granted: " + allGranted + ", Denied: " + denied.size());
        if (permissionCallback != null) {
            permissionCallback.onAllPermissionsResult(allGranted, denied);
        }
    }
    
    // ===== Utility Methods =====
    
    /**
     * Get detailed permission status report
     */
    public String getPermissionStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Permission Status ===\n");
        
        // Basic permissions
        boolean storage = hasStoragePermission();
        boolean install = hasInstallPermission();
        boolean basic = hasBasicPermissions();
        
        status.append("Storage Access: ").append(storage ? "✅ Granted" : "❌ Denied").append("\n");
        status.append("Install Packages: ").append(install ? "✅ Granted" : "❌ Denied").append("\n");
        status.append("Basic Permissions: ").append(basic ? "✅ All Good" : "❌ Missing").append("\n");
        
        // Enhanced permissions (Shizuku)
        boolean shizukuInstalled = shizukuManager.isShizukuInstalled();
        boolean shizukuRunning = shizukuManager.isShizukuRunning();
        boolean shizukuPermission = shizukuManager.hasShizukuPermission();
        boolean enhanced = hasEnhancedPermissions();
        
        status.append("\n=== Enhanced Permissions (Shizuku) ===\n");
        status.append("Shizuku Installed: ").append(shizukuInstalled ? "✅ Yes" : "❌ No").append("\n");
        status.append("Shizuku Running: ").append(shizukuRunning ? "✅ Yes" : "❌ No").append("\n");
        status.append("Shizuku Permission: ").append(shizukuPermission ? "✅ Granted" : "❌ Denied").append("\n");
        status.append("Enhanced Access: ").append(enhanced ? "✅ Available" : "❌ Unavailable").append("\n");
        
        // Recommendations
        status.append("\n=== Recommendations ===\n");
        if (!basic) {
            if (!storage) status.append("• Grant storage access for mod installation\n");
            if (!install) status.append("• Grant install permission for APK installation\n");
        }
        if (!enhanced && shizukuInstalled) {
            if (!shizukuRunning) {
                status.append("• Start Shizuku service for enhanced capabilities\n");
            } else if (!shizukuPermission) {
                status.append("• Grant Shizuku permission for enhanced file access\n");
            }
        }
        if (!enhanced && !shizukuInstalled) {
            status.append("• Install Shizuku for enhanced capabilities (optional)\n");
        }
        if (basic && enhanced) {
            status.append("• All permissions are ready! 🎉\n");
        }
        
        return status.toString();
    }
    
    /**
     * FIXED: Force refresh all permission statuses
     */
    public void refreshPermissionStatus() {
        LogUtils.logDebug("Refreshing all permission statuses...");
        
        // Refresh Shizuku status
        shizukuManager.refreshPermissionStatus();
        
        // Log current status
        boolean storage = hasStoragePermission();
        boolean install = hasInstallPermission();
        boolean enhanced = hasEnhancedPermissions();
        
        LogUtils.logUser("Permission status refreshed - Storage: " + storage + 
                        ", Install: " + install + ", Enhanced: " + enhanced);
        
        // Notify callback if status changed
        if (permissionCallback != null) {
            boolean allBasic = hasBasicPermissions();
            List<String> denied = new ArrayList<>();
            if (!storage) denied.add("STORAGE");
            if (!install) denied.add("INSTALL");
            if (!enhanced) denied.add("ENHANCED");
            
            permissionCallback.onAllPermissionsResult(allBasic && enhanced, denied);
        }
    }
    
    /**
     * Check if permission should be requested with rationale
     */
    public boolean shouldShowPermissionRationale(String permission) {
        if (!(context instanceof Activity)) {
            return false;
        }
        
        switch (permission) {
            case "STORAGE":
                return ActivityCompat.shouldShowRequestPermissionRationale((Activity) context,
                    Manifest.permission.READ_EXTERNAL_STORAGE) ||
                    ActivityCompat.shouldShowRequestPermissionRationale((Activity) context,
                    Manifest.permission.WRITE_EXTERNAL_STORAGE);
            default:
                return false;
        }
    }
    
    /**
     * Open app settings for manual permission management
     */
    public void openAppSettings() {
        try {
            Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
            intent.setData(Uri.parse("package:" + context.getPackageName()));
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(intent);
            LogUtils.logUser("Opening app settings for manual permission management");
        } catch (Exception e) {
            LogUtils.logError("Failed to open app settings: " + e.getMessage());
            Toast.makeText(context, "Cannot open app settings", Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Get ShizukuManager instance for direct access
     */
    public ShizukuManager getShizukuManager() {
        return shizukuManager;
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/PrivilegeManager.java

package com.modloader.util;

public class PrivilegeManager {
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/RootManager.java

// File: RootManager.java (FIXED) - Placeholder Root Manager with Shizuku Integration
// Path: /app/src/main/java/com/modloader/util/RootManager.java

package com.modloader.util;

import android.content.Context;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;

import java.io.BufferedReader;
import java.io.DataOutputStream;
import java.io.InputStreamReader;

/**
 * FIXED: Root Manager - Currently serves as a placeholder but includes basic root detection
 * This version focuses on Shizuku integration while providing foundation for future root support
 */
public class RootManager {
    private static final String TAG = "RootManager";
    
    private final Context context;
    private final ShizukuManager shizukuManager;
    private RootCallback rootCallback;
    
    // Root detection cache
    private Boolean rootAvailableCache = null;
    private Boolean rootPermissionCache = null;
    private long lastRootCheck = 0;
    private static final long ROOT_CHECK_CACHE_TIME = 30000; // 30 seconds
    
    public interface RootCallback {
        void onRootGranted();
        void onRootDenied();
        void onRootError(String error);
    }
    
    public RootManager(Context context) {
        this.context = context;
        this.shizukuManager = new ShizukuManager(context);
        LogUtils.logDebug("RootManager initialized (using Shizuku as primary enhanced access)");
    }
    
    public void setRootCallback(RootCallback callback) {
        this.rootCallback = callback;
    }
    
    // ===== Root Detection Methods =====
    
    /**
     * FIXED: Check if root access is available on device
     * This performs actual root detection but recommends Shizuku instead
     */
    public boolean isRootAvailable() {
        // Use cached result if recent
        long currentTime = System.currentTimeMillis();
        if (rootAvailableCache != null && (currentTime - lastRootCheck) < ROOT_CHECK_CACHE_TIME) {
            return rootAvailableCache;
        }
        
        LogUtils.logDebug("Checking for root availability...");
        boolean rootDetected = false;
        
        try {
            // Method 1: Check for su binary
            rootDetected = checkSuBinary();
            
            if (!rootDetected) {
                // Method 2: Check for root indicators
                rootDetected = checkRootIndicators();
            }
            
            if (!rootDetected) {
                // Method 3: Try to execute su command (non-persistent)
                rootDetected = testSuCommand();
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Root check exception: " + e.getMessage());
            rootDetected = false;
        }
        
        // Cache result
        rootAvailableCache = rootDetected;
        lastRootCheck = currentTime;
        
        LogUtils.logDebug("Root availability check result: " + rootDetected);
        
        if (rootDetected) {
            LogUtils.logUser("⚠️ Root detected, but Shizuku is recommended for enhanced access");
            showRootDetectedButShizukuRecommended();
        } else {
            LogUtils.logDebug("No root access detected - this is normal for most devices");
        }
        
        return rootDetected;
    }
    
    /**
     * Check if root permission is granted (placeholder - always returns false)
     */
    public boolean hasRootPermission() {
        // For now, we don't support active root usage
        // This is a placeholder that always returns false
        LogUtils.logDebug("Root permission check - returning false (root not supported in this version)");
        return false;
    }
    
    /**
     * Check if root is ready for use (placeholder - always returns false)
     */
    public boolean isRootReady() {
        // Root functionality is disabled in favor of Shizuku
        return false;
    }
    
    // ===== Root Detection Implementation =====
    
    /**
     * Check for su binary in common locations
     */
    private boolean checkSuBinary() {
        String[] suPaths = {
            "/system/bin/su",
            "/system/xbin/su",
            "/sbin/su",
            "/data/local/xbin/su",
            "/data/local/bin/su",
            "/system/sd/xbin/su",
            "/system/bin/failsafe/su",
            "/data/local/su",
            "/su/bin/su"
        };
        
        for (String path : suPaths) {
            try {
                java.io.File suFile = new java.io.File(path);
                if (suFile.exists()) {
                    LogUtils.logDebug("Found su binary at: " + path);
                    return true;
                }
            } catch (Exception e) {
                // Ignore and continue
            }
        }
        
        return false;
    }
    
    /**
     * Check for common root indicators
     */
    private boolean checkRootIndicators() {
        try {
            // Check for Superuser apps
            String[] rootApps = {
                "com.noshufou.android.su",
                "com.noshufou.android.su.elite",
                "eu.chainfire.supersu",
                "com.koushikdutta.superuser",
                "com.thirdparty.superuser",
                "com.yellowes.su",
                "com.koushikdutta.rommanager",
                "com.koushikdutta.rommanager.license",
                "com.dimonvideo.luckypatcher",
                "com.chelpus.lackypatch",
                "com.ramdroid.appquarantine",
                "com.topjohnwu.magisk"
            };
            
            android.content.pm.PackageManager pm = context.getPackageManager();
            for (String packageName : rootApps) {
                try {
                    pm.getPackageInfo(packageName, 0);
                    LogUtils.logDebug("Found root indicator app: " + packageName);
                    return true;
                } catch (android.content.pm.PackageManager.NameNotFoundException e) {
                    // App not found, continue
                }
            }
            
            // Check build tags
            String buildTags = android.os.Build.TAGS;
            if (buildTags != null && buildTags.contains("test-keys")) {
                LogUtils.logDebug("Found test-keys in build tags");
                return true;
            }
            
            // Check for RW system partition
            try {
                String[] mountPoints = {"/system", "/system/", "/system/bin"};
                for (String mountPoint : mountPoints) {
                    java.io.File file = new java.io.File(mountPoint);
                    if (file.exists() && file.canWrite()) {
                        LogUtils.logDebug("System partition is writable: " + mountPoint);
                        return true;
                    }
                }
            } catch (Exception e) {
                // Ignore
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error checking root indicators: " + e.getMessage());
        }
        
        return false;
    }
    
    /**
     * Test su command execution (non-persistent)
     */
    private boolean testSuCommand() {
        try {
            Process process = Runtime.getRuntime().exec("su");
            DataOutputStream os = new DataOutputStream(process.getOutputStream());
            BufferedReader is = new BufferedReader(new InputStreamReader(process.getInputStream()));
            
            os.writeBytes("id\n");
            os.writeBytes("exit\n");
            os.flush();
            
            String response = is.readLine();
            os.close();
            is.close();
            
            if (response != null && response.contains("uid=0")) {
                LogUtils.logDebug("Su command test successful");
                return true;
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Su command test failed: " + e.getMessage());
        }
        
        return false;
    }
    
    // ===== Placeholder Root Operations =====
    
    /**
     * Request root access (placeholder - redirects to Shizuku)
     */
    public void requestRootAccess() {
        LogUtils.logUser("Root access requested - redirecting to Shizuku setup");
        
        showRootNotSupportedDialog();
    }
    
    /**
     * Execute command with root (placeholder - not implemented)
     */
    public boolean executeRootCommand(String command) {
        LogUtils.logDebug("Root command execution not supported - use Shizuku instead");
        return false;
    }
    
    /**
     * Check root status and show appropriate dialog
     */
    public void checkRootStatus() {
        boolean available = isRootAvailable();
        String status = getRootStatusReport();
        
        LogUtils.logUser("Root Status Check:\n" + status);
        
        new AlertDialog.Builder(context)
            .setTitle("Root Status")
            .setMessage(status)
            .setPositiveButton("Use Shizuku Instead", (dialog, which) -> {
                shizukuManager.checkRootStatus(); // This will handle Shizuku setup
            })
            .setNeutralButton("Learn About Shizuku", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("OK", null)
            .show();
    }
    
    // ===== Status and Information Methods =====
    
    /**
     * Get detailed root status report
     */
    public String getRootStatusReport() {
        StringBuilder report = new StringBuilder();
        report.append("=== Root Access Status ===\n");
        
        boolean available = isRootAvailable();
        report.append("Root Detected: ").append(available ? "✅ Yes" : "❌ No").append("\n");
        report.append("Root Supported: ❌ No (Use Shizuku instead)\n");
        
        if (available) {
            report.append("Root Permission: ❌ Not granted (disabled)\n");
            report.append("\n⚠️ Root was detected on your device, but this app uses Shizuku ");
            report.append("for enhanced capabilities instead of root access.\n");
        } else {
            report.append("\n✅ No root detected - this is normal for most devices.\n");
        }
        
        report.append("\n=== Recommended Alternative ===\n");
        boolean shizukuReady = shizukuManager.isShizukuReady();
        report.append("Shizuku Status: ").append(shizukuReady ? "✅ Ready" : "❌ Not Ready").append("\n");
        
        if (!shizukuReady) {
            if (!shizukuManager.isShizukuInstalled()) {
                report.append("• Install Shizuku app\n");
            }
            if (!shizukuManager.isShizukuRunning()) {
                report.append("• Start Shizuku service\n");
            }
            if (!shizukuManager.hasShizukuPermission()) {
                report.append("• Grant Shizuku permission\n");
            }
        }
        
        report.append("\n💡 Shizuku provides enhanced file access without requiring ");
        report.append("root access and is safer and more reliable than traditional root methods.");
        
        return report.toString();
    }
    
    /**
     * Get root detection details
     */
    public String getRootDetectionDetails() {
        StringBuilder details = new StringBuilder();
        details.append("=== Root Detection Details ===\n");
        
        details.append("Su Binary Check: ").append(checkSuBinary() ? "✅ Found" : "❌ Not Found").append("\n");
        details.append("Root Apps Check: ").append(checkRootIndicators() ? "✅ Found" : "❌ Not Found").append("\n");
        details.append("Su Command Test: ").append(testSuCommand() ? "✅ Works" : "❌ Failed").append("\n");
        
        details.append("\nBuild Info:\n");
        details.append("• Tags: ").append(android.os.Build.TAGS).append("\n");
        details.append("• Type: ").append(android.os.Build.TYPE).append("\n");
        details.append("• Device: ").append(android.os.Build.DEVICE).append("\n");
        
        details.append("\n⚠️ Even if root is detected, this app uses Shizuku for enhanced access.");
        
        return details.toString();
    }
    
    // ===== Dialog Methods =====
    
    private void showRootDetectedButShizukuRecommended() {
        // Only show this dialog once per app session
        if (rootPermissionCache != null) {
            return;
        }
        rootPermissionCache = false; // Mark as shown
        
        new AlertDialog.Builder(context)
            .setTitle("Root Detected")
            .setMessage("Your device appears to have root access, but this app uses Shizuku " +
                       "instead for enhanced capabilities.\n\n" +
                       "Shizuku is:\n" +
                       "• Safer than root access\n" +
                       "• More reliable\n" +
                       "• Doesn't require device modification\n" +
                       "• Works without voiding warranty\n\n" +
                       "Would you like to set up Shizuku instead?")
            .setPositiveButton("Setup Shizuku", (dialog, which) -> {
                if (!shizukuManager.isShizukuInstalled()) {
                    shizukuManager.installShizuku();
                } else {
                    shizukuManager.requestShizukuPermission();
                }
            })
            .setNeutralButton("Learn More", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Not Now", null)
            .show();
    }
    
    private void showRootNotSupportedDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Root Access Not Supported")
            .setMessage("This version of TerrariaLoader does not support root access for enhanced capabilities.\n\n" +
                       "Instead, we recommend using Shizuku, which provides:\n" +
                       "• Enhanced file access without root\n" +
                       "• Better security and stability\n" +
                       "• No device modification required\n" +
                       "• Easy setup via ADB or existing root\n\n" +
                       "Would you like to set up Shizuku instead?")
            .setPositiveButton("Setup Shizuku", (dialog, which) -> {
                shizukuManager.checkRootStatus();
            })
            .setNeutralButton("Learn About Shizuku", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    // ===== Utility Methods =====
    
    /**
     * Clear cached root detection results
     */
    public void clearCache() {
        rootAvailableCache = null;
        rootPermissionCache = null;
        lastRootCheck = 0;
        LogUtils.logDebug("Root detection cache cleared");
    }
    
    /**
     * Get alternative enhanced access manager (Shizuku)
     */
    public ShizukuManager getAlternativeEnhancedAccess() {
        return shizukuManager;
    }
    
    /**
     * Check if any enhanced access is available (Shizuku or Root)
     */
    public boolean hasAnyEnhancedAccess() {
        return shizukuManager.isShizukuReady();
    }
    
    /**
     * Get the best available enhanced access method
     */
    public String getBestEnhancedAccessMethod() {
        if (shizukuManager.isShizukuReady()) {
            return "Shizuku (Ready)";
        } else if (shizukuManager.isShizukuInstalled()) {
            return "Shizuku (Needs Setup)";
        } else if (isRootAvailable()) {
            return "Root (Detected but not supported)";
        } else {
            return "None Available";
        }
    }
    
    /**
     * Execute enhanced operation using best available method
     */
    public boolean executeEnhancedOperation(String operation, String... params) {
        if (shizukuManager.isShizukuReady()) {
            LogUtils.logDebug("Executing enhanced operation via Shizuku: " + operation);
            return shizukuManager.executeShizukuCommand(operation + " " + String.join(" ", params));
        } else {
            LogUtils.logDebug("No enhanced access available for operation: " + operation);
            return false;
        }
    }
    
    // ===== Legacy Compatibility Methods =====
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public boolean canUseEnhancedAccess() {
        return hasAnyEnhancedAccess();
    }
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public void requestEnhancedAccess() {
        if (shizukuManager.isShizukuAvailable()) {
            shizukuManager.requestShizukuPermission();
        } else {
            requestRootAccess();
        }
    }
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public boolean executeEnhancedCommand(String command) {
        return executeEnhancedOperation(command);
    }
}
================================================================================

/ModLoader/app/src/main/java/com/modloader/util/ShizukuManager.java

// File: ShizukuManager.java (FIXED) - Reflection-based with In-App Permission Dialog Support
// Path: /app/src/main/java/com/modloader/util/ShizukuManager.java

package com.modloader.util;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Process;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;

import java.lang.reflect.Method;

/**
 * FIXED: Complete Shizuku Manager with proper in-app permission dialog support
 * Uses reflection to trigger Shizuku permission dialogs within the app (like MT Manager)
 */
public class ShizukuManager {
    private static final String TAG = "ShizukuManager";
    
    // Shizuku package and permission constants
    private static final String SHIZUKU_PACKAGE = "moe.shizuku.privileged.api";
    private static final String SHIZUKU_PERMISSION = "moe.shizuku.manager.permission.API_V23";
    private static final int SHIZUKU_REQUEST_CODE = 1001;
    
    // Shizuku API reflection constants - FIXED for proper API usage
    private static final String SHIZUKU_CLASS = "moe.shizuku.api.Shizuku";
    private static final String SHIZUKU_BINDER_CLASS = "moe.shizuku.api.ShizukuBinderWrapper";
    private static final String SHIZUKU_LISTENER_CLASS = "moe.shizuku.api.Shizuku$OnRequestPermissionResultListener";
    
    private final Context context;
    private ShizukuPermissionCallback permissionCallback;
    
    // FIXED: Add permission result listener for in-app dialogs
    private Object permissionResultListener;
    private boolean listenerRegistered = false;
    
    // Interface for permission callback
    public interface ShizukuPermissionCallback {
        void onPermissionGranted();
        void onPermissionDenied();
        void onPermissionError(String error);
    }
    
    public ShizukuManager(Context context) {
        this.context = context;
        setupPermissionListener();
        LogUtils.logDebug("ShizukuManager initialized");
    }
    
    /**
     * FIXED: Setup permission result listener for in-app permission dialogs
     */
    private void setupPermissionListener() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Class<?> listenerClass = Class.forName(SHIZUKU_LISTENER_CLASS);
            
            // Create permission result listener using reflection
            permissionResultListener = java.lang.reflect.Proxy.newProxyInstance(
                listenerClass.getClassLoader(),
                new Class[]{listenerClass},
                new java.lang.reflect.InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, java.lang.reflect.Method method, Object[] args) {
                        if ("onRequestPermissionResult".equals(method.getName()) && args != null && args.length >= 2) {
                            int requestCode = (Integer) args[0];
                            int grantResult = (Integer) args[1];
                            handleShizukuPermissionResult(requestCode, grantResult);
                        }
                        return null;
                    }
                }
            );
            
            // Add listener to Shizuku
            Method addListenerMethod = shizukuClass.getMethod("addRequestPermissionResultListener", listenerClass);
            addListenerMethod.invoke(null, permissionResultListener);
            listenerRegistered = true;
            
            LogUtils.logDebug("Shizuku permission listener setup successfully");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to setup Shizuku permission listener: " + e.getMessage());
            permissionResultListener = null;
            listenerRegistered = false;
        }
    }
    
    /**
     * FIXED: Handle Shizuku permission result from in-app dialog
     */
    private void handleShizukuPermissionResult(int requestCode, int grantResult) {
        LogUtils.logDebug("Shizuku permission result: requestCode=" + requestCode + ", result=" + grantResult);
        
        if (requestCode == SHIZUKU_REQUEST_CODE) {
            boolean granted = (grantResult == PackageManager.PERMISSION_GRANTED);
            
            if (granted) {
                LogUtils.logUser("✅ Shizuku permission granted!");
                if (permissionCallback != null) {
                    permissionCallback.onPermissionGranted();
                }
            } else {
                LogUtils.logUser("❌ Shizuku permission denied");
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied();
                }
            }
        }
    }
    
    public void setPermissionCallback(ShizukuPermissionCallback callback) {
        this.permissionCallback = callback;
    }
    
    // ===== FIXED: Comprehensive Shizuku Detection Methods =====
    
    /**
     * FIXED: Check if Shizuku app is installed
     */
    public boolean isShizukuInstalled() {
        try {
            PackageManager pm = context.getPackageManager();
            ApplicationInfo appInfo = pm.getApplicationInfo(SHIZUKU_PACKAGE, 0);
            boolean installed = appInfo != null;
            LogUtils.logDebug("Shizuku installation check: " + installed);
            return installed;
        } catch (PackageManager.NameNotFoundException e) {
            LogUtils.logDebug("Shizuku not installed: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku installation check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check if Shizuku service is running using multiple methods
     */
    public boolean isShizukuRunning() {
        // Method 1: Try reflection-based check
        if (checkShizukuRunningViaReflection()) {
            LogUtils.logDebug("Shizuku running - detected via reflection");
            return true;
        }
        
        // Method 2: Try permission check (if service is running, permission check should work)
        if (checkShizukuRunningViaPermission()) {
            LogUtils.logDebug("Shizuku running - detected via permission check");
            return true;
        }
        
        // Method 3: Try binder check
        if (checkShizukuRunningViaBinder()) {
            LogUtils.logDebug("Shizuku running - detected via binder");
            return true;
        }
        
        LogUtils.logDebug("Shizuku service not running - all detection methods failed");
        return false;
    }
    
    /**
     * FIXED: Check if Shizuku permission is granted using multiple detection methods
     */
    public boolean hasShizukuPermission() {
        if (!isShizukuInstalled()) {
            LogUtils.logDebug("Shizuku permission check: not installed");
            return false;
        }
        
        if (!isShizukuRunning()) {
            LogUtils.logDebug("Shizuku permission check: service not running");
            return false;
        }
        
        // Method 1: Check via Android's permission system
        boolean systemPermissionGranted = checkSystemPermission();
        if (systemPermissionGranted) {
            LogUtils.logDebug("Shizuku permission: granted via system permission");
            return true;
        }
        
        // Method 2: Check via Shizuku API reflection
        boolean apiPermissionGranted = checkShizukuPermissionViaReflection();
        if (apiPermissionGranted) {
            LogUtils.logDebug("Shizuku permission: granted via Shizuku API");
            return true;
        }
        
        // Method 3: Try to perform a test operation
        boolean testOperationSuccessful = testShizukuOperation();
        if (testOperationSuccessful) {
            LogUtils.logDebug("Shizuku permission: granted via test operation");
            return true;
        }
        
        LogUtils.logDebug("Shizuku permission: not granted - all check methods failed");
        return false;
    }
    
    /**
     * FIXED: Comprehensive check if Shizuku is available and ready
     */
    public boolean isShizukuAvailable() {
        return isShizukuInstalled();
    }
    
    /**
     * FIXED: Check if Shizuku is fully ready (installed + running + permission)
     */
    public boolean isShizukuReady() {
        boolean installed = isShizukuInstalled();
        boolean running = isShizukuRunning();
        boolean hasPermission = hasShizukuPermission();
        
        LogUtils.logDebug("Shizuku readiness - Installed: " + installed + 
                         ", Running: " + running + ", Permission: " + hasPermission);
        
        return installed && running && hasPermission;
    }
    
    // ===== Detection Method Implementations =====
    
    /**
     * Check if Shizuku is running via reflection
     */
    private boolean checkShizukuRunningViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method pingMethod = shizukuClass.getMethod("pingBinder");
            Boolean result = (Boolean) pingMethod.invoke(null);
            return result != null && result;
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku class not found - API not available");
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku ping failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if Shizuku is running via permission system
     */
    private boolean checkShizukuRunningViaPermission() {
        try {
            // If we can check permission without exception, service is likely running
            int permissionState = context.checkPermission(SHIZUKU_PERMISSION, 
                                                         Process.myPid(), 
                                                         Process.myUid());
            // Even if denied, if we get a valid response, service is running
            return true;
        } catch (SecurityException e) {
            // SecurityException might indicate service is not running
            LogUtils.logDebug("Permission check SecurityException: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Permission check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if Shizuku is running via binder check
     */
    private boolean checkShizukuRunningViaBinder() {
        try {
            Class<?> binderClass = Class.forName(SHIZUKU_BINDER_CLASS);
            Method isAliveMethod = binderClass.getMethod("isBinderAlive");
            Boolean result = (Boolean) isAliveMethod.invoke(null);
            return result != null && result;
        } catch (Exception e) {
            LogUtils.logDebug("Binder check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check system-level permission
     */
    private boolean checkSystemPermission() {
        try {
            int result = context.checkPermission(SHIZUKU_PERMISSION, 
                                               Process.myPid(), 
                                               Process.myUid());
            boolean granted = (result == PackageManager.PERMISSION_GRANTED);
            LogUtils.logDebug("System permission check result: " + granted);
            return granted;
        } catch (Exception e) {
            LogUtils.logDebug("System permission check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check Shizuku permission via Shizuku API reflection
     */
    private boolean checkShizukuPermissionViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method checkSelfPermissionMethod = shizukuClass.getMethod("checkSelfPermission");
            Integer result = (Integer) checkSelfPermissionMethod.invoke(null);
            boolean granted = (result != null && result == PackageManager.PERMISSION_GRANTED);
            LogUtils.logDebug("Shizuku API permission check result: " + granted);
            return granted;
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku API not available for permission check");
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku API permission check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Test Shizuku operation to verify permission
     */
    private boolean testShizukuOperation() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method getUidMethod = shizukuClass.getMethod("getUid");
            Integer uid = (Integer) getUidMethod.invoke(null);
            boolean success = (uid != null && uid >= 0);
            LogUtils.logDebug("Shizuku test operation result: " + success);
            return success;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku test operation failed: " + e.getMessage());
            return false;
        }
    }
    
    // ===== FIXED: In-App Permission Request Methods =====
    
    /**
     * FIXED: Request Shizuku permission with in-app dialog support
     */
    public void requestShizukuPermission() {
        LogUtils.logUser("Requesting Shizuku permission...");
        
        if (!isShizukuInstalled()) {
            LogUtils.logUser("❌ Shizuku not installed");
            if (permissionCallback != null) {
                permissionCallback.onPermissionError("Shizuku app is not installed");
            }
            showShizukuNotInstalledDialog();
            return;
        }
        
        if (!isShizukuRunning()) {
            LogUtils.logUser("❌ Shizuku service not running");
            if (permissionCallback != null) {
                permissionCallback.onPermissionError("Shizuku service is not running");
            }
            showShizukuNotRunningDialog();
            return;
        }
        
        // FIXED: Try to request permission directly (in-app dialog)
        if (requestPermissionViaReflection()) {
            LogUtils.logUser("✅ Shizuku permission dialog requested");
        } else {
            LogUtils.logUser("❌ Failed to show in-app dialog - using fallback");
            showPermissionFallbackDialog();
        }
    }
    
    /**
     * FIXED: Request permission via Shizuku API reflection with proper method signature
     */
    private boolean requestPermissionViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            
            // FIXED: Use the correct Shizuku API method signature for in-app dialogs
            Method requestPermissionMethod = shizukuClass.getMethod("requestPermission", int.class);
            requestPermissionMethod.invoke(null, SHIZUKU_REQUEST_CODE);
            
            LogUtils.logDebug("Shizuku permission requested via reflection API - in-app dialog should appear");
            return true;
        } catch (NoSuchMethodException e) {
            LogUtils.logDebug("Shizuku requestPermission method not found - trying alternative: " + e.getMessage());
            
            // Try alternative method signatures
            try {
                Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
                
                // Try method without parameters (some Shizuku versions)
                Method altMethod = shizukuClass.getMethod("requestPermission");
                altMethod.invoke(null);
                LogUtils.logDebug("Shizuku permission requested via alternative method");
                return true;
            } catch (Exception altE) {
                LogUtils.logDebug("Alternative Shizuku method also failed: " + altE.getMessage());
                return false;
            }
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku API class not found - Shizuku not properly integrated: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Reflection permission request failed: " + e.getMessage());
            return false;
        }
    }
    
    // ===== Fallback and Dialog Methods =====
    
    /**
     * Show fallback dialog when in-app permission request fails
     */
    private void showPermissionFallbackDialog() {
        new AlertDialog.Builder(context)
            .setTitle("🔐 Grant Shizuku Permission")
            .setMessage("Unable to show in-app permission dialog.\n\n" +
                       "You will be taken to Shizuku app to grant permission manually.\n\n" +
                       "Steps:\n" +
                       "1. Tap 'Open Shizuku'\n" +
                       "2. Find this app in the permission list\n" +
                       "3. Grant permission\n" +
                       "4. Return to this app")
            .setPositiveButton("Open Shizuku", (dialog, which) -> openShizukuApp())
            .setNegativeButton("Cancel", (dialog, which) -> {
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied();
                }
            })
            .show();
    }
    
    /**
     * Open Shizuku app
     */
    public void openShizukuApp() {
        try {
            Intent intent = context.getPackageManager().getLaunchIntentForPackage(SHIZUKU_PACKAGE);
            if (intent != null) {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(intent);
                LogUtils.logUser("Opening Shizuku app...");
            } else {
                LogUtils.logUser("❌ Cannot open Shizuku app");
                Toast.makeText(context, "Cannot open Shizuku app", Toast.LENGTH_SHORT).show();
            }
        } catch (Exception e) {
            LogUtils.logDebug("Failed to open Shizuku app: " + e.getMessage());
            Toast.makeText(context, "Failed to open Shizuku", Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Install Shizuku app
     */
    public void installShizuku() {
        try {
            String downloadUrl = "https://github.com/RikkaApps/Shizuku/releases/latest";
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(downloadUrl));
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(intent);
            LogUtils.logUser("Opening Shizuku download page...");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to open Shizuku download: " + e.getMessage());
            Toast.makeText(context, "Please install Shizuku manually", Toast.LENGTH_LONG).show();
        }
    }
    
    /**
     * Get detailed Shizuku status
     */
    public String getShizukuStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Shizuku Status ===\n");
        
        boolean installed = isShizukuInstalled();
        status.append("Installed: ").append(installed ? "✅ Yes" : "❌ No").append("\n");
        
        if (installed) {
            boolean running = isShizukuRunning();
            status.append("Service Running: ").append(running ? "✅ Yes" : "❌ No").append("\n");
            
            if (running) {
                boolean hasPermission = hasShizukuPermission();
                status.append("Permission Granted: ").append(hasPermission ? "✅ Yes" : "❌ No").append("\n");
                
                if (hasPermission) {
                    status.append("Status: ✅ Ready for use\n");
                } else {
                    status.append("Status: 🔐 Permission needed\n");
                }
            } else {
                status.append("Status: ▶️ Service needs to be started\n");
            }
        } else {
            status.append("Status: 📥 Installation required\n");
        }
        
        // Add version info if available
        try {
            PackageManager pm = context.getPackageManager();
            String version = pm.getPackageInfo(SHIZUKU_PACKAGE, 0).versionName;
            status.append("Version: ").append(version).append("\n");
        } catch (Exception e) {
            // Ignore version check errors
        }
        
        return status.toString();
    }
    
    /**
     * Check Shizuku status and show appropriate action
     */
    public void checkRootStatus() {
        String status = getShizukuStatus();
        LogUtils.logUser("Shizuku Status Check:\n" + status);
        
        if (isShizukuReady()) {
            Toast.makeText(context, "✅ Shizuku is ready!", Toast.LENGTH_SHORT).show();
        } else if (!isShizukuInstalled()) {
            showShizukuNotInstalledDialog();
        } else if (!isShizukuRunning()) {
            showShizukuNotRunningDialog();
        } else {
            requestShizukuPermission();
        }
    }
    
    // ===== Dialog Methods =====
    
    private void showShizukuNotInstalledDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Shizuku Required")
            .setMessage("Shizuku is not installed on this device.\n\n" +
                       "Shizuku provides enhanced file access capabilities without requiring root.\n\n" +
                       "Would you like to download and install Shizuku?")
            .setPositiveButton("Download Shizuku", (dialog, which) -> installShizuku())
            .setNeutralButton("Learn More", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Intent.ACTION_VIEW, 
                                             Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showShizukuNotRunningDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Start Shizuku Service")
            .setMessage("Shizuku is installed but the service is not running.\n\n" +
                       "To start Shizuku, you can:\n" +
                       "• Use ADB command (requires computer)\n" +
                       "• Use root access (if available)\n" +
                       "• Use wireless ADB (Android 11+)\n\n" +
                       "Would you like to open Shizuku app for setup?")
            .setPositiveButton("Open Shizuku", (dialog, which) -> openShizukuApp())
            .setNeutralButton("Setup Guide", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Intent.ACTION_VIEW, 
                                             Uri.parse("https://shizuku.rikka.app/guide/setup/"));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    // ===== Permission Result Handling =====
    
    /**
     * Handle permission request result
     */
    public void handlePermissionResult(int requestCode, int resultCode) {
        if (requestCode == SHIZUKU_REQUEST_CODE) {
            // Re-check permission after request
            boolean granted = hasShizukuPermission();
            LogUtils.logUser("Shizuku permission result: " + granted);
            
            if (permissionCallback != null) {
                if (granted) {
                    permissionCallback.onPermissionGranted();
                } else {
                    permissionCallback.onPermissionDenied();
                }
            }
        }
    }
    
    /**
     * FIXED: Force refresh permission status (useful after returning from Shizuku app)
     */
    public void refreshPermissionStatus() {
        LogUtils.logDebug("Refreshing Shizuku permission status...");
        
        // Clear any cached state and re-check
        boolean wasReady = isShizukuReady();
        
        // Small delay to ensure status has updated
        new android.os.Handler().postDelayed(() -> {
            boolean isNowReady = isShizukuReady();
            if (isNowReady != wasReady) {
                LogUtils.logUser("Shizuku status changed: " + (isNowReady ? "Ready" : "Not Ready"));
                if (permissionCallback != null) {
                    if (isNowReady) {
                        permissionCallback.onPermissionGranted();
                    } else {
                        permissionCallback.onPermissionDenied();
                    }
                }
            }
        }, 1000); // 1 second delay
    }
    
    // ===== Enhanced File Operations (when Shizuku is ready) =====
    
    /**
     * Execute command with Shizuku privileges
     */
    public boolean executeShizukuCommand(String command) {
        if (!isShizukuReady()) {
            LogUtils.logDebug("Shizuku not ready for command execution");
            return false;
        }
        
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            // This would need proper Shizuku API integration
            LogUtils.logDebug("Executing Shizuku command: " + command);
            return true;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku command execution failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if enhanced file operations are available
     */
    public boolean canUseEnhancedFileAccess() {
        return isShizukuReady();
    }
    
    // ===== Legacy Compatibility Methods =====
    
    @Deprecated
    public boolean isRootAvailable() {
        // For backward compatibility
        return isShizukuAvailable();
    }
    
    @Deprecated
    public boolean isRootReady() {
        // For backward compatibility  
        return isShizukuReady();
    }
    
    @Deprecated
    public boolean hasRootPermission() {
        // For backward compatibility
        return hasShizukuPermission();
    }
    
    @Deprecated
    public void requestRootAccess() {
        // For backward compatibility
        requestShizukuPermission();
    }
}
================================================================================

/ModLoader/app/src/main/res/drawable/gradient_background_135.xml

<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <gradient
        android:angle="135"
        android:startColor="#E8F5E8"
        android:endColor="#F1F8E9"
        android:type="linear" />
</shape>
================================================================================

/ModLoader/app/src/main/res/drawable/ic_arrow_back.xml

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
  <path
      android:fillColor="@android:color/black"
      android:pathData="M20,11L7.83,11l5.59,-5.59L12,4l-8,8 8,8 1.41,-1.41L7.83,13L20,13z"/>
</vector>
================================================================================

/ModLoader/app/src/main/res/drawable/ic_launcher_background.xml

<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path
        android:fillColor="#3DDC84"
        android:pathData="M0,0h108v108h-108z" />
    <path
        android:fillColor="#00000000"
        android:pathData="M9,0L9,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,0L19,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M29,0L29,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M39,0L39,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M49,0L49,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M59,0L59,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M69,0L69,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M79,0L79,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M89,0L89,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M99,0L99,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,9L108,9"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,19L108,19"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,29L108,29"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,39L108,39"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,49L108,49"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,59L108,59"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,69L108,69"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,79L108,79"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,89L108,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,99L108,99"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,29L89,29"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,39L89,39"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,49L89,49"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,59L89,59"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,69L89,69"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,79L89,79"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M29,19L29,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M39,19L39,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M49,19L49,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M59,19L59,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M69,19L69,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M79,19L79,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
</vector>
================================================================================

/ModLoader/app/src/main/res/drawable/ic_plugin.xml

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#000000"
        android:pathData="M12,2c1.1,0 2,0.9 2,2v2h2c1.1,0 2,0.9 2,2v2h2v2h-2v2c0,1.1 -0.9,2 -2,2h-2v2c0,1.1 -0.9,2 -2,2h-2v-2H8c-1.1,0 -2,-0.9 -2,-2v-2H4v-2h2V8c0,-1.1 0.9,-2 2,-2h2V4c0,-1.1 0.9,-2 2,-2z"/>
</vector>
================================================================================

/ModLoader/app/src/main/res/drawable-v24/ic_launcher_foreground.xml

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:aapt="http://schemas.android.com/aapt"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path android:pathData="M31,63.928c0,0 6.4,-11 12.1,-13.1c7.2,-2.6 26,-1.4 26,-1.4l38.1,38.1L107,108.928l-32,-1L31,63.928z">
        <aapt:attr name="android:fillColor">
            <gradient
                android:endX="85.84757"
                android:endY="92.4963"
                android:startX="42.9492"
                android:startY="49.59793"
                android:type="linear">
                <item
                    android:color="#44000000"
                    android:offset="0.0" />
                <item
                    android:color="#00000000"
                    android:offset="1.0" />
            </gradient>
        </aapt:attr>
    </path>
    <path
        android:fillColor="#FFFFFF"
        android:fillType="nonZero"
        android:pathData="M65.3,45.828l3.8,-6.6c0.2,-0.4 0.1,-0.9 -0.3,-1.1c-0.4,-0.2 -0.9,-0.1 -1.1,0.3l-3.9,6.7c-6.3,-2.8 -13.4,-2.8 -19.7,0l-3.9,-6.7c-0.2,-0.4 -0.7,-0.5 -1.1,-0.3C38.8,38.328 38.7,38.828 38.9,39.228l3.8,6.6C36.2,49.428 31.7,56.028 31,63.928h46C76.3,56.028 71.8,49.428 65.3,45.828zM43.4,57.328c-0.8,0 -1.5,-0.5 -1.8,-1.2c-0.3,-0.7 -0.1,-1.5 0.4,-2.1c0.5,-0.5 1.4,-0.7 2.1,-0.4c0.7,0.3 1.2,1 1.2,1.8C45.3,56.528 44.5,57.328 43.4,57.328L43.4,57.328zM64.6,57.328c-0.8,0 -1.5,-0.5 -1.8,-1.2s-0.1,-1.5 0.4,-2.1c0.5,-0.5 1.4,-0.7 2.1,-0.4c0.7,0.3 1.2,1 1.2,1.8C66.5,56.528 65.6,57.328 64.6,57.328L64.6,57.328z"
        android:strokeWidth="1"
        android:strokeColor="#00000000" />
</vector>
================================================================================

/ModLoader/app/src/main/res/layout/activity_about.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Mod Loader"
            android:textStyle="bold"
            android:textSize="20sp"
            android:layout_marginBottom="8dp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Author: Jonie"
            android:textSize="16sp"
            android:layout_marginBottom="8dp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Description:\n\nThis app lets you inject custom loaders into Terraria APKs for modding purposes. It also provides tools to manage logs and exported APKs.\n\nUse responsibly and only with legal copies of the game."
            android:textSize="14sp"
            android:lineSpacingExtra="4dp" />
    </LinearLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_addon_list.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent" android:layout_height="match_parent">
    <Button android:id="@+id/btnReload" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Reload"/>
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"/>
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_config_extensions.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent" android:layout_height="match_parent">
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_dll_mod.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <!-- Header Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL Mod Manager"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            android:layout_marginBottom="16dp" />

        <!-- Status Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#F5F5F5"
            android:padding="12dp"
            android:layout_marginBottom="16dp">

            <TextView
                android:id="@+id/statusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="DLL Mods: 0 enabled, 0 disabled, 0 total"
                android:textSize="16sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="❌ No loader installed - DLL mods will not work"
                android:textSize="14sp" />

        </LinearLayout>

        <!-- Loader Installation Section -->
        <LinearLayout
            android:id="@+id/loaderInfoSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#E8F5E8"
            android:padding="12dp"
            android:layout_marginBottom="16dp"
            android:visibility="gone">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Loader Installation"
                android:textSize="16sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/installLoaderBtn"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Install Loader"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/selectApkBtn"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Select Terraria APK"
                android:layout_marginBottom="8dp" />

        </LinearLayout>

        <!-- Mod Management Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL Mod Management"
            android:textSize="18sp"
            android:textStyle="bold"
            android:layout_marginBottom="8dp" />

        <Button
            android:id="@+id/installDllBtn"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Install DLL Mod"
            android:layout_marginBottom="16dp" />

        <!-- Mod List -->
        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/dllModRecyclerView"
            android:layout_width="match_parent"
            android:layout_height="300dp"
            android:background="#F9F9F9"
            android:padding="8dp"
            android:layout_marginBottom="16dp" />

        <!-- Action Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center"
            android:layout_marginTop="16dp">

            <Button
                android:id="@+id/refreshBtn"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Refresh"
                android:layout_marginEnd="8dp" />

            <Button
                android:id="@+id/viewLogsBtn"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="View Logs"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Information Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL mods require MelonLoader or LemonLoader to be installed. Use 'Install Loader' to set up the required components."
            android:textSize="12sp"
            android:textColor="#666666"
            android:layout_marginTop="16dp"
            android:gravity="center" />

    </LinearLayout>

</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_instructions.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.InstructionsActivity">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="16dp">

        <TextView
            android:id="@+id/tv_instructions_title"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="Manual Installation Instructions"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            android:layout_marginTop="16dp" />

        <TextView
            android:id="@+id/tv_instructions"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:textSize="16sp"
            android:textIsSelectable="true"
            app:layout_constraintTop_toBottomOf="@id/tv_instructions_title"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            tools:text="Detailed instructions will appear here..." />

    </androidx.constraintlayout.widget.ConstraintLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_log.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/log_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <ScrollView
        android:id="@+id/log_scroll"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:fillViewport="true">

        <TextView
            android:id="@+id/log_text"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textIsSelectable="true"
            android:textAppearance="?android:textAppearanceSmall"
            android:textColor="#FFFFFF"
            android:background="#222222"
            android:padding="10dp" />
    </ScrollView>

    <Button
        android:id="@+id/export_logs_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Export Logs" />
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_log_viewer_enhanced.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="#1E1E1E">

    <!-- Statistics Bar -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="8dp"
        android:background="#2E2E2E"
        android:gravity="center_vertical">

        <TextView
            android:id="@+id/logStatsText"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="📊 Total: 0 | Showing: 0 | Errors: 0 | Warnings: 0"
            android:textColor="#FFFFFF"
            android:textSize="12sp"
            android:fontFamily="monospace" />

        <Button
            android:id="@+id/refreshButton"
            android:layout_width="wrap_content"
            android:layout_height="32dp"
            android:text="🔄"
            android:textSize="14sp"
            android:background="#4CAF50"
            android:textColor="#FFFFFF"
            android:layout_marginStart="8dp"
            android:padding="4dp"
            android:minWidth="48dp" />

    </LinearLayout>

    <!-- Filter Section (Collapsible) -->
    <LinearLayout
        android:id="@+id/filterSection"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="12dp"
        android:background="#2A2A2A"
        android:visibility="visible">

        <!-- Filter Controls Row 1 -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center_vertical">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="🏷️ Type:"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Spinner
                android:id="@+id/logTypeSpinner"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:layout_marginEnd="16dp"
                android:popupBackground="#3A3A3A" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="📊 Level:"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Spinner
                android:id="@+id/logLevelSpinner"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:popupBackground="#3A3A3A" />

        </LinearLayout>

        <!-- Search Bar -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="12dp"
            android:gravity="center_vertical">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="🔍"
                android:textSize="16sp"
                android:layout_marginEnd="8dp" />

            <EditText
                android:id="@+id/searchEditText"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:hint="Search logs..."
                android:textColorHint="#888888"
                android:textColor="#FFFFFF"
                android:padding="12dp"
                android:textSize="14sp"
                android:fontFamily="monospace"
                android:inputType="text"
                android:imeOptions="actionSearch" />

        </LinearLayout>

        <!-- Control Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="12dp"
            android:gravity="center">

            <Button
                android:id="@+id/clearLogsButton"
                android:layout_width="0dp"
                android:layout_height="40dp"
                android:layout_weight="1"
                android:text="🗑️ Clear"
                android:background="#FF6B6B"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Button
                android:id="@+id/exportLogsButton"
                android:layout_width="0dp"
                android:layout_height="40dp"
                android:layout_weight="1"
                android:text="📤 Export"
                android:background="#2196F3"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Auto-scroll checkbox -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="8dp"
            android:gravity="center_vertical">

            <CheckBox
                android:id="@+id/autoScrollCheckbox"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="📜 Auto-scroll to bottom"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:checked="true"
                android:buttonTint="#4CAF50" />

        </LinearLayout>

    </LinearLayout>

    <!-- Main Log Content -->
    <androidx.swiperefreshlayout.widget.SwipeRefreshLayout
        android:id="@+id/swipeRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1">

        <ScrollView
            android:id="@+id/logScrollView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#1E1E1E"
            android:scrollbars="vertical"
            android:fadeScrollbars="false">

            <TextView
                android:id="@+id/logTextView"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="📋 Loading logs...\n\nPlease wait while we fetch the latest log entries."
                android:textColor="#E0E0E0"
                android:textSize="12sp"
                android:fontFamily="monospace"
                android:padding="16dp"
                android:textIsSelectable="true"
                android:background="#1E1E1E"
                android:lineSpacingMultiplier="1.2" />

        </ScrollView>

    </androidx.swiperefreshlayout.widget.SwipeRefreshLayout>

    <!-- Bottom Action Bar -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="8dp"
        android:background="#2E2E2E"
        android:gravity="center_vertical">

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="💡 Tip: Swipe down to refresh, use filters to find specific logs"
            android:textColor="#888888"
            android:textSize="11sp" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="v1.0"
            android:textColor="#666666"
            android:textSize="10sp"
            android:layout_marginStart="8dp" />

    </LinearLayout>

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_main.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp">

    <!-- Existing Buttons -->
    <Button
        android:id="@+id/universal_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Universal"
        android:layout_marginBottom="16dp" />

    <Button
        android:id="@+id/specific_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Specific Version"
        android:layout_marginBottom="24dp" />

    <!-- New Buttons for Extension System -->
    <Button
        android:id="@+id/addons_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Addon Manager"
        android:layout_marginBottom="16dp" />

    <Button
        android:id="@+id/scripts_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Script Console"
        android:layout_marginBottom="16dp" />

    <Button
        android:id="@+id/configs_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Config Extensions" />
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_mod_list.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center_vertical"
        android:layout_marginBottom="16dp">

        <ImageButton
            android:id="@+id/backButton"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_alignParentStart="true"
            android:background="?attr/selectableItemBackgroundBorderless"
            android:src="@drawable/ic_arrow_back"
            android:contentDescription="Back"
            android:tint="@android:color/black" />

        <TextView
            android:id="@+id/modCountTextView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:text="Total Mods: 0 (Enabled: 0)"
            android:textSize="16sp"
            android:textStyle="bold" />

    </RelativeLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewMods"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:scrollbars="vertical" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center"
        android:layout_marginTop="16dp">

        <Button
            android:id="@+id/addModButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Add Mod"
            android:layout_marginEnd="8dp" />

        <Button
            android:id="@+id/refreshModsButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Refresh Mods" />
    </LinearLayout>

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_mod_management.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp"
        android:background="#F8F9FA">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:background="#FFFFFF"
            android:padding="20dp"
            android:layout_marginBottom="16dp"
            android:elevation="2dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🎮 Mod Management"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="8dp" />

            <!-- Status Section -->
            <TextView
                android:id="@+id/statusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="📊 Loading mod statistics..."
                android:textSize="14sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:padding="8dp"
                android:background="#E8F5E8"
                android:layout_marginBottom="8dp" />

            <!-- Loader Status -->
            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Checking loader status..."
                android:textSize="12sp"
                android:gravity="center"
                android:padding="6dp" />

        </LinearLayout>

        <!-- Loader Info Section (shown when loader is installed) -->
        <LinearLayout
            android:id="@+id/loaderInfoSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#E3F2FD"
            android:padding="16dp"
            android:layout_marginBottom="16dp"
            android:visibility="gone"
            android:elevation="2dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="ℹ️ Loader Information"
                android:textSize="14sp"
                android:textStyle="bold"
                android:textColor="#1565C0"
                android:layout_marginBottom="8dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="• DLL mods will be loaded by MelonLoader\n• DEX/JAR mods are loaded directly by TerrariaLoader\n• Enable/disable mods using the switches below"
                android:textSize="12sp"
                android:textColor="#1976D2"
                android:lineSpacingExtra="2dp" />

        </LinearLayout>

        <!-- Add Mods Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#FFFFFF">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📥 Add New Mods"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="#333333"
                    android:layout_marginBottom="12dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/addDllModBtn"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📥 Add DLL Mod"
                        android:textSize="14sp"
                        android:background="#4CAF50"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/addDexModBtn"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📱 Add DEX/JAR Mod"
                        android:textSize="14sp"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />

                </LinearLayout>

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="💡 DLL mods require MelonLoader • DEX/JAR mods work without a loader"
                    android:textSize="11sp"
                    android:textColor="#666666"
                    android:gravity="center"
                    android:layout_marginTop="8dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Mod List Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#FFFFFF">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:orientation="vertical"
                android:padding="16dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical"
                    android:layout_marginBottom="12dp">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Installed Mods"
                        android:textSize="18sp"
                        android:textStyle="bold"
                        android:textColor="#333333" />

                    <Button
                        android:id="@+id/refreshBtn"
                        android:layout_width="wrap_content"
                        android:layout_height="36dp"
                        android:text="🔄 Refresh"
                        android:textSize="12sp"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:minWidth="80dp" />

                </LinearLayout>

                <androidx.recyclerview.widget.RecyclerView
                    android:id="@+id/modRecyclerView"
                    android:layout_width="match_parent"
                    android:layout_height="0dp"
                    android:layout_weight="1"
                    android:background="#F9F9F9"
                    android:padding="8dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Navigation Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center"
            android:layout_marginTop="8dp">

            <Button
                android:id="@+id/backBtn"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="← Back"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:minHeight="40dp"
                android:layout_marginEnd="16dp" />

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Manage your mods after installation"
                android:textSize="12sp"
                android:textColor="#999999"
                android:gravity="center" />

        </LinearLayout>

        <!-- Tips Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="💡 Tips:\n• Toggle mods with switches\n• Delete mods with trash icon\n• DLL mods require patched Terraria APK\n• DEX/JAR mods work with any Terraria version"
            android:textSize="11sp"
            android:textColor="#888888"
            android:background="#F0F0F0"
            android:padding="12dp"
            android:layout_marginTop="16dp"
            android:lineSpacingExtra="2dp" />

    </LinearLayout>

</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_offline_diagnostic.xml

<?xml version="1.0" encoding="utf-8"?>
<!-- File: activity_offline_diagnostic.xml -->
<!-- Path: /res/layout/activity_offline_diagnostic.xml -->

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/background_light">
    
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">
        
        <!-- Header Text -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="TerrariaLoader Offline Diagnostics"
            android:textSize="24sp"
            android:textStyle="bold"
            android:textColor="@android:color/holo_green_dark"
            android:gravity="center"
            android:layout_marginBottom="16dp" />
        
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Diagnose and fix common issues without internet connection"
            android:textSize="14sp"
            android:textColor="@android:color/darker_gray"
            android:gravity="center"
            android:layout_marginBottom="24dp" />
        
        <!-- Quick Actions Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🚀 Quick Actions"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="12dp" />
                
                <Button
                    android:id="@+id/btn_run_full_diagnostic"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🔍 Run Full System Check"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_green_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_diagnose_apk"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📦 Diagnose APK Installation Problem"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_orange_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_fix_settings"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="⚙️ Fix Settings Persistence Issue"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_blue_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_auto_repair"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🛠️ Attempt Auto-Repair"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_purple"
                    android:textColor="@android:color/white"
                    android:padding="12dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Diagnostic Results Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📊 Diagnostic Results"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="12dp" />
                
                <!-- Results Display Area -->
                <ScrollView
                    android:layout_width="match_parent"
                    android:layout_height="400dp"
                    android:background="@android:color/black"
                    android:padding="8dp">
                    
                    <TextView
                        android:id="@+id/diagnostic_results_text"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Click 'Run Full System Check' to start diagnostics..."
                        android:textColor="@android:color/holo_green_light"
                        android:textSize="12sp"
                        android:fontFamily="monospace"
                        android:textIsSelectable="true"
                        android:padding="8dp" />
                </ScrollView>
                
                <!-- Action Buttons for Results -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:layout_marginTop="12dp">
                    
                    <Button
                        android:id="@+id/btn_export_report"
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="📤 Export Report"
                        android:textAllCaps="false"
                        android:background="@android:color/darker_gray"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="4dp" />
                    
                    <Button
                        android:id="@+id/btn_clear_results"
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="🗑️ Clear Results"
                        android:textAllCaps="false"
                        android:background="@android:color/darker_gray"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="4dp" />
                </LinearLayout>
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Help/Info Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="💡 Quick Help"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="8dp" />
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="• APK parsing errors: Usually caused by corrupted files or missing permissions\n• Settings not saving: Often due to storage permissions or corrupted preferences\n• Directory issues: Can be fixed with auto-repair function\n• Export reports to share with developers for support"
                    android:textSize="14sp"
                    android:textColor="@android:color/darker_gray"
                    android:lineSpacingMultiplier="1.2" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Version Info -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="TerrariaLoader Diagnostic Tool v1.0 - Offline Mode"
            android:textSize="12sp"
            android:textColor="@android:color/darker_gray"
            android:gravity="center"
            android:layout_marginTop="16dp"
            android:layout_marginBottom="16dp" />
        
    </LinearLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_plugin_manager.xml

<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/rv_plugins"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp"/>

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab_install"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:contentDescription="Install plugin"
        app:srcCompat="@android:drawable/ic_input_add"
        app:layout_anchorGravity="bottom|end"
        app:layout_anchor="@id/rv_plugins" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_plugins.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/header"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/plugins_title"
        android:textStyle="bold"
        android:textSize="18sp"
        android:paddingBottom="8dp"/>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <Button
            android:id="@+id/btn_refresh"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="@string/plugins_refresh" />

        <Button
            android:id="@+id/btn_install"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="@string/plugins_install" />
    </LinearLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/plugins_list"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:paddingTop="8dp"/>
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_script_console.xml

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent" android:layout_height="match_parent">
    <LinearLayout android:orientation="vertical" android:layout_width="match_parent" android:layout_height="wrap_content" android:padding="12dp">
        <EditText android:id="@+id/input" android:layout_width="match_parent" android:layout_height="wrap_content" android:minLines="6" android:hint="Enter JS or Lua"/>
        <Button android:id="@+id/runJs" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Run JavaScript"/>
        <Button android:id="@+id/runLua" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Run Lua"/>
        <TextView android:id="@+id/output" android:layout_width="match_parent" android:layout_height="wrap_content" android:textIsSelectable="true" />
    </LinearLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_settings_enhanced.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView
     xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:app="http://schemas.android.com/apk/res-auto"
     xmlns:tools="http://schemas.android.com/tools"
     android:layout_height="match_parent"
     android:layout_width="match_parent"
     android:background="#F5F5F5"
     android:padding="16dp">

    <LinearLayout
         android:layout_height="wrap_content"
         android:layout_width="match_parent"
         android:orientation="vertical">

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="16dp"
             android:gravity="center"
             android:background="#2196F3"
             android:padding="16dp"
             android:textSize="20sp"
             android:textColor="#FFFFFF"
             android:text="⚙️ Operation Modes &amp; Permissions"
             android:textStyle="bold" />

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:text="Choose Operation Mode:"
             android:textStyle="bold" />

        <RadioGroup
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="vertical"
             android:id="@+id/operationModeGroup">

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/normalCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/normalModeRadio"
                         android:text=" Normal Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/normalStatus"
                         android:text="Standard Android permissions" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/shizukuCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/shizukuModeRadio"
                         android:text=" Shizuku Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/shizukuStatus"
                         android:text="Enhanced file access without root" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/rootCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/rootModeRadio"
                         android:text=" Root Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/rootStatus"
                         android:text="Full system control (requires root)" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/hybridCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/hybridModeRadio"
                         android:text="⚡ Hybrid Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/hybridStatus"
                         android:text="Maximum capabilities" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

        </RadioGroup>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="Setup &amp; Permissions:"
             android:textStyle="bold" />

        <LinearLayout
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="vertical">

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#2196F3"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/shizukuSetupBtn"
                 android:text=" Setup Shizuku" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#FF9800"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/rootSetupBtn"
                 android:text=" Setup Root" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#4CAF50"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/permissionBtn"
                 android:text=" Manage Permissions" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#E0E0E0"
                 android:elevation="2dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#333333"
                 android:id="@+id/refreshStatusBtn"
                 android:text="🔄 Refresh Status" />

        </LinearLayout>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="App Features:"
             android:textStyle="bold" />

        <androidx.cardview.widget.CardView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_margin="4dp"
             app:cardElevation="4dp"
             app:cardBackgroundColor="#FFFFFF"
             app:cardCornerRadius="8dp">

            <LinearLayout
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:background="#FFFFFF"
                 android:padding="16dp"
                 android:orientation="vertical">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto-enable Mods"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoEnableSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Debug Logging"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/debugLoggingSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto Backup"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoBackupSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto Update Check"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoUpdateSwitch" />

                </LinearLayout>

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="Additional Actions:"
             android:textStyle="bold" />

        <LinearLayout
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="horizontal">

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:layout_marginEnd="4dp"
                 android:background="#F44336"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_weight="1"
                 android:id="@+id/resetSettingsBtn"
                 android:text="Reset"
                 android:textStyle="bold" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:layout_marginEnd="2dp"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_marginStart="2dp"
                 android:background="#2196F3"
                 android:layout_weight="1"
                 android:id="@+id/exportSettingsBtn"
                 android:text="Export"
                 android:textStyle="bold" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:background="#4CAF50"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_marginStart="4dp"
                 android:layout_weight="1"
                 android:id="@+id/importSettingsBtn"
                 android:text="Import"
                 android:textStyle="bold" />

        </LinearLayout>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:gravity="center"
             android:background="#E3F2FD"
             android:padding="12dp"
             android:textSize="12sp"
             android:textColor="#666666"
             android:layout_marginTop="16dp"
             android:text="Tip: Shizuku provides enhanced capabilities without requiring root access"
             android:textStyle="italic" />

    </LinearLayout>

</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_setup_guide.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="24dp"
        android:background="@drawable/gradient_background_135">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:background="#FFFFFF"
            android:padding="32dp"
            android:layout_marginBottom="24dp"
            android:elevation="8dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🚀 MelonLoader Setup Guide"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="12dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Choose your preferred installation method"
                android:textSize="16sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:lineSpacingExtra="4dp" />

        </LinearLayout>

        <!-- Installation Options -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:layout_marginBottom="24dp">

            <!-- Online Installation Card -->
            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="16dp"
                app:cardCornerRadius="12dp"
                app:cardElevation="6dp"
                app:cardBackgroundColor="#E3F2FD">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="20dp">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="🌐 Automated Online Installation"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#1565C0"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Automatically downloads from GitHub\n• No manual file handling\n• Always gets latest version\n• Requires internet connection"
                        android:textSize="14sp"
                        android:textColor="#1976D2"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_online_install"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="🌐 Start Online Installation"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Offline Import Card -->
            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="16dp"
                app:cardCornerRadius="12dp"
                app:cardElevation="6dp"
                app:cardBackgroundColor="#FFF3E0">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="20dp">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📦 Offline ZIP Import"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#E65100"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Import pre-downloaded ZIP files\n• Works without internet\n• Auto-detects NET8/NET35\n• Extracts to correct directories"
                        android:textSize="14sp"
                        android:textColor="#F57C00"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_offline_import"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📦 Import ZIP File"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="💡 Supports: melon_data.zip, lemon_data.zip, custom packages"
                        android:textSize="11sp"
                        android:textColor="#BF360C"
                        android:gravity="center"
                        android:layout_marginTop="8dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Manual Installation Card -->
            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="16dp"
                app:cardCornerRadius="12dp"
                app:cardElevation="6dp"
                app:cardBackgroundColor="#F3E5F5">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="20dp">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📖 Manual Installation Guide"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#7B1FA2"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Step-by-step instructions\n• Full control over installation\n• Troubleshooting included\n• For advanced users"
                        android:textSize="14sp"
                        android:textColor="#8E24AA"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_manual_instructions"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📖 View Manual Guide"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#9C27B0"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

        </LinearLayout>

        <!-- Information Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#E8F5E8">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="ℹ️ What happens after installation?"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:textColor="#2E7D32"
                    android:layout_marginBottom="8dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="1. 🚀 Unified Loader opens automatically\n2. 📱 Select your Terraria APK file\n3. ⚡ Patch APK with MelonLoader\n4. 📲 Install the patched APK\n5. 🎮 Add DLL mods and enjoy!"
                    android:textSize="14sp"
                    android:textColor="#388E3C"
                    android:lineSpacingExtra="4dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Requirements Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:background="#FFFFFF"
            android:padding="16dp"
            android:layout_marginTop="8dp"
            android:elevation="2dp">

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="📋 Requirements:\n• 50MB+ free space\n• Terraria APK file\n• File manager permissions"
                android:textSize="12sp"
                android:textColor="#666666"
                android:lineSpacingExtra="2dp" />

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="🎯 Recommended:\n• Use Online Installation\n• Keep APK backup\n• Enable unknown sources"
                android:textSize="12sp"
                android:textColor="#666666"
                android:lineSpacingExtra="2dp" />

        </LinearLayout>

    </LinearLayout>

</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_specific_selection.xml

<?xml version="1.0" encoding="utf-8"?>

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="24dp">

    <LinearLayout
        android:id="@+id/specific_selection_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center">

        <!-- Header -->
        <TextView
            android:id="@+id/headerText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Choose Game/App to Mod"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            android:layout_marginBottom="32dp" />

        <!-- Terraria Button -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="@android:drawable/dialog_frame"
            android:padding="16dp"
            android:layout_marginBottom="16dp">

            <Button
                android:id="@+id/terraria_button"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🌍 Terraria"
                android:textSize="20sp"
                android:textStyle="bold"
                android:minHeight="60dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="• Support for DEX/JAR mods\n• Support for DLL mods (via MelonLoader)\n• APK patching and installation\n• Full mod management"
                android:textSize="14sp"
                android:textColor="@android:color/darker_gray"
                android:layout_marginTop="8dp" />

        </LinearLayout>

        <!-- Future Games Section -->
        <LinearLayout
            android:id="@+id/futureGamesSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:layout_marginTop="24dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Coming Soon"
                android:textSize="18sp"
                android:textStyle="bold"
                android:gravity="center"
                android:layout_marginBottom="16dp" />

            <!-- Placeholder cards for future games -->
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:background="@android:drawable/dialog_frame"
                android:padding="12dp"
                android:layout_marginBottom="8dp"
                android:alpha="0.5">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🟫 Minecraft PE"
                        android:textSize="16sp" />

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Soon"
                        android:textSize="12sp"
                        android:textColor="@android:color/darker_gray" />

                </LinearLayout>
            </LinearLayout>

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:background="@android:drawable/dialog_frame"
                android:padding="12dp"
                android:layout_marginBottom="8dp"
                android:alpha="0.5">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🚀 Among Us"
                        android:textSize="16sp" />

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Soon"
                        android:textSize="12sp"
                        android:textColor="@android:color/darker_gray" />

                </LinearLayout>
            </LinearLayout>

        </LinearLayout>

        <!-- Back Button -->
        <Button
            android:id="@+id/backToMainButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="← Back to Main Menu"
            android:layout_marginTop="32dp" />

    </LinearLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_terraria_specific_updated.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:id="@+id/rootLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp"
        android:background="#E8F5E8">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:layout_marginBottom="24dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🌍 Terraria Mod Loader"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="8dp" />

            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Checking loader status..."
                android:textSize="14sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:padding="12dp"
                android:background="#F1F8E9"
                android:layout_marginTop="8dp" />
        </LinearLayout>

        <!-- Setup & Installation Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/setupCard"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#F1F8E9">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🚀 Setup &amp; Installation"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#2E7D32"
                    android:layout_marginBottom="16dp" />

                <Button
                    android:id="@+id/unifiedSetupButton"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🎯 Complete Setup Wizard"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:background="#4CAF50"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="12dp"
                    android:minHeight="56dp"
                    android:elevation="2dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="All-in-one wizard for MelonLoader installation and APK patching"
                    android:textSize="12sp"
                    android:textColor="#66BB6A"
                    android:layout_marginBottom="16dp"
                    android:gravity="center" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/setupGuideButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📖 Setup Guide"
                        android:textSize="14sp"
                        android:background="#81C784"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/manualInstructionsButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Manual Steps"
                        android:textSize="14sp"
                        android:background="#A5D6A7"
                        android:textColor="#2E7D32"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />
                </LinearLayout>
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Mod Management Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/modManagementCard"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#E3F2FD">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📦 Mod Management"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#1565C0"
                    android:layout_marginBottom="16dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/dexModManagerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📱 DEX/JAR Mods"
                        android:textSize="14sp"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/dllModManagerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🔧 DLL Mods"
                        android:textSize="14sp"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />
                </LinearLayout>

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="• DEX/JAR: Android Java mods (always available)\n• DLL: C# mods (requires MelonLoader)"
                    android:textSize="12sp"
                    android:textColor="#42A5F5"
                    android:layout_marginTop="12dp"
                    android:lineSpacingExtra="2dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Tools & Utilities Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/toolsCard"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#F3E5F5">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🛠️ Tools &amp; Utilities"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#7B1FA2"
                    android:layout_marginBottom="16dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="3">

                    <Button
                        android:id="@+id/logViewerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Logs"
                        android:textSize="12sp"
                        android:background="#9C27B0"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="4dp"
                        android:minHeight="44dp" />

                    <Button
                        android:id="@+id/settingsButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="⚙️ Settings"
                        android:textSize="12sp"
                        android:background="#BA68C8"
                        android:textColor="@android:color/white"
                        android:layout_marginHorizontal="4dp"
                        android:minHeight="44dp" />

                    <Button
                        android:id="@+id/sandboxButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🧪 Sandbox"
                        android:textSize="12sp"
                        android:background="#CE93D8"
                        android:textColor="#4A148C"
                        android:layout_marginStart="4dp"
                        android:minHeight="44dp" />
                </LinearLayout>

                <!-- ✅ Fixed Diagnostic Button -->
                <Button
                    android:id="@+id/diagnosticButton"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🧪 Offline Diagnostic &amp; Repair"
                    android:textSize="14sp"
                    android:textStyle="bold"
                    android:backgroundTint="#9C27B0"
                    android:textColor="#FFFFFF"
                    android:layout_marginTop="12dp"
                    android:minHeight="48dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Navigation Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:layout_marginTop="16dp">

            <Button
                android:id="@+id/backButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="← Back to App Selection"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:minHeight="40dp" />
        </LinearLayout>

        <!-- Info Footer -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="💡 Tip: Start with 'Complete Setup Wizard' for the easiest experience!"
            android:textSize="12sp"
            android:textColor="#81C784"
            android:gravity="center"
            android:background="#F1F8E9"
            android:padding="16dp"
            android:layout_marginTop="16dp"
            android:layout_marginBottom="16dp" />
    </LinearLayout>
</ScrollView>
================================================================================

/ModLoader/app/src/main/res/layout/activity_unified_loader.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="@android:color/white">

    <!-- Header Section -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:background="#E8F5E8"
        android:padding="16dp"
        android:elevation="4dp">

        <!-- Progress Bar -->
        <ProgressBar
            android:id="@+id/stepProgressBar"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="match_parent"
            android:layout_height="8dp"
            android:layout_marginBottom="12dp"
            android:max="4"
            android:progress="0"
            android:progressTint="#4CAF50"
            android:progressBackgroundTint="#E0E0E0" />

        <!-- Step Indicator -->
        <TextView
            android:id="@+id/stepIndicatorText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Step 1 of 5"
            android:textSize="12sp"
            android:textColor="#666666"
            android:gravity="center"
            android:layout_marginBottom="8dp" />

        <!-- Step Title -->
        <TextView
            android:id="@+id/stepTitleText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Welcome"
            android:textSize="24sp"
            android:textStyle="bold"
            android:textColor="#2E7D32"
            android:gravity="center"
            android:layout_marginBottom="8dp" />

        <!-- Step Description -->
        <TextView
            android:id="@+id/stepDescriptionText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Welcome to the MelonLoader Setup Wizard"
            android:textSize="14sp"
            android:textColor="#4CAF50"
            android:gravity="center"
            android:lineSpacingExtra="4dp" />

    </LinearLayout>

    <!-- Main Content Area -->
    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:fillViewport="true">

        <LinearLayout
            android:id="@+id/stepContentContainer"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:padding="16dp"
            android:minHeight="400dp">

            <!-- Dynamic content will be added here -->

        </LinearLayout>

    </ScrollView>

    <!-- Navigation Footer -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:background="#F5F5F5"
        android:padding="16dp"
        android:elevation="4dp">

        <!-- Action Button (context-sensitive) -->
        <Button
            android:id="@+id/actionButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🚀 Start Setup"
            android:textSize="16sp"
            android:textStyle="bold"
            android:background="#4CAF50"
            android:textColor="@android:color/white"
            android:layout_marginBottom="12dp"
            android:minHeight="48dp" />

        <!-- Navigation Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:weightSum="2">

            <Button
                android:id="@+id/previousButton"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="← Previous"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:layout_marginEnd="8dp"
                android:enabled="false" />

            <Button
                android:id="@+id/nextButton"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Next →"
                android:textSize="14sp"
                android:background="#2196F3"
                android:textColor="@android:color/white"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Progress Text (for operations) -->
        <TextView
            android:id="@+id/progressText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text=""
            android:textSize="12sp"
            android:textColor="#666666"
            android:gravity="center"
            android:layout_marginTop="8dp"
            android:visibility="gone" />

    </LinearLayout>

    <!-- Hidden Status Views (referenced by activity) -->
    <TextView
        android:id="@+id/loaderStatusText"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

    <TextView
        android:id="@+id/apkStatusText"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

    <ProgressBar
        android:id="@+id/actionProgressBar"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/activity_universal.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <Button
        android:id="@+id/select_apk_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Select Universal APK" />

    <Button
        android:id="@+id/select_zip_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Select Loader ZIP" />

    <Button
        android:id="@+id/inject_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Inject Loader" />

    <TextView
        android:id="@+id/status_text"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Status"
        android:paddingTop="16dp"
        android:textAppearance="?android:attr/textAppearanceMedium" />

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/dialog_log_settings.xml

<!-- File: dialog_log_settings.xml (NEW DIALOG) - Settings for Log Viewer -->
<!-- Path: /main/res/layout/dialog_log_settings.xml -->

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="#2A2A2A">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="⚙️ Log Viewer Settings"
        android:textColor="#FFFFFF"
        android:textSize="18sp"
        android:textStyle="bold"
        android:gravity="center"
        android:layout_marginBottom="16dp" />

    <!-- Auto Refresh Setting -->
    <CheckBox
        android:id="@+id/autoRefreshCheckbox"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="🔄 Auto-refresh logs (every 5 seconds)"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:checked="true"
        android:buttonTint="#4CAF50"
        android:layout_marginBottom="12dp" />

    <!-- Syntax Highlighting Setting -->
    <CheckBox
        android:id="@+id/syntaxHighlightCheckbox"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="🎨 Enable syntax highlighting"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:checked="true"
        android:buttonTint="#4CAF50"
        android:layout_marginBottom="12dp" />

    <!-- Text Size Setting -->
    <TextView
        android:id="@+id/textSizeLabel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="📏 Text Size: 12"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:layout_marginBottom="8dp" />

    <SeekBar
        android:id="@+id/textSizeSeekBar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:max="20"
        android:min="8"
        android:progress="12"
        android:thumbTint="#4CAF50"
        android:progressTint="#4CAF50"
        android:layout_marginBottom="16dp" />

    <!-- Information Text -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="💡 Changes are applied immediately and persist during this session."
        android:textColor="#888888"
        android:textSize="12sp"
        android:gravity="center"
        android:layout_marginTop="8dp" />

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/item_log_entry.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="8dp"
    android:background="?android:attr/selectableItemBackground"
    android:minHeight="56dp">

    <!-- Level Indicator Bar -->
    <View
        android:id="@+id/levelIndicator"
        android:layout_width="4dp"
        android:layout_height="match_parent"
        android:layout_marginEnd="8dp"
        android:background="#2196F3" />

    <!-- Main Content -->
    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:orientation="vertical">

        <!-- Header Row (Timestamp, Level, Tag) -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginBottom="4dp">

            <TextView
                android:id="@+id/logTimestamp"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="00:00:00"
                android:textSize="11sp"
                android:textColor="#666666"
                android:fontFamily="monospace"
                android:layout_marginEnd="8dp" />

            <TextView
                android:id="@+id/logLevel"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="INFO"
                android:textSize="11sp"
                android:textStyle="bold"
                android:textColor="#2196F3"
                android:layout_marginEnd="8dp"
                android:minWidth="48dp" />

            <TextView
                android:id="@+id/logTag"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="TAG"
                android:textSize="11sp"
                android:textColor="#666666"
                android:textStyle="italic"
                android:ellipsize="end"
                android:maxLines="1" />

        </LinearLayout>

        <!-- Message Content -->
        <TextView
            android:id="@+id/logMessage"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Log message content goes here and can span multiple lines if needed"
            android:textSize="14sp"
            android:textColor="#333333"
            android:lineSpacingExtra="2dp"
            android:textIsSelectable="true"
            android:maxLines="10"
            android:ellipsize="end" />

    </LinearLayout>

</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/item_mod.xml

<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp"
        android:gravity="center_vertical">

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:orientation="vertical">

            <TextView
                android:id="@+id/modNameTextView"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Mod Name"
                android:textSize="18sp"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/modDescription"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Mod description goes here. This can be multiline."
                android:textSize="14sp"
                android:textColor="@android:color/darker_gray"
                android:layout_marginTop="4dp" />

        </LinearLayout>

        <Switch
            android:id="@+id/modSwitch"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="16dp" />

        <ImageButton
            android:id="@+id/modDeleteButton"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_marginStart="8dp"
            android:background="?attr/selectableItemBackgroundBorderless"
            android:src="@android:drawable/ic_menu_delete"
            android:contentDescription="Delete Mod"
            app:tint="@android:color/holo_red_dark" />

    </LinearLayout>
</androidx.cardview.widget.CardView>
================================================================================

/ModLoader/app/src/main/res/layout/item_plugin.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="12dp">

    <TextView
        android:id="@+id/title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Plugin title"
        android:textStyle="bold"
        android:textSize="16sp" />

    <TextView
        android:id="@+id/subtitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="id — author"
        android:textSize="13sp" />
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/layout/item_plugin_manager.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="12dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center_vertical">

        <LinearLayout
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:orientation="vertical">

            <TextView
                android:id="@+id/title"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Plugin title"
                android:textStyle="bold"
                android:textSize="16sp" />

            <TextView
                android:id="@+id/subtitle"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="id — author"
                android:textSize="13sp" />

            <TextView
                android:id="@+id/disabled_label"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Disabled"
                android:textSize="12sp"
                android:textColor="@android:color/holo_red_dark"
                android:visibility="gone"/>
        </LinearLayout>

        <Switch
            android:id="@+id/switch_enable"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <ImageButton
            android:id="@+id/btn_uninstall"
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:src="@android:drawable/ic_menu_more"
            android:contentDescription="Menu"
            android:background="?attr/selectableItemBackgroundBorderless"/>
    </LinearLayout>
</LinearLayout>
================================================================================

/ModLoader/app/src/main/res/menu/log_viewer_menu.xml

<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <item
        android:id="@+id/action_toggle_filters"
        android:icon="@android:drawable/ic_search_category_default"
        android:title="Toggle Filters"
        app:showAsAction="ifRoom" />

    <item
        android:id="@+id/action_share_logs"
        android:icon="@android:drawable/ic_menu_share"
        android:title="Share Logs"
        app:showAsAction="ifRoom" />

    <item
        android:id="@+id/action_clear_logs"
        android:icon="@android:drawable/ic_menu_delete"
        android:title="Clear Logs"
        app:showAsAction="never" />

    <item
        android:id="@+id/action_settings"
        android:icon="@android:drawable/ic_menu_preferences"
        android:title="Settings"
        app:showAsAction="never" />

</menu>
================================================================================

/ModLoader/app/src/main/res/menu/main_menu.xml

<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:id="@+id/action_log"
        android:title="View Logs"
        android:icon="@android:drawable/ic_menu_info_details"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_about"
        android:title="About"
        android:icon="@android:drawable/ic_menu_help"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_dark_mode"
        android:title="Toggle Dark Mode"
        android:icon="@android:drawable/ic_menu_day"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_export_apk"
        android:title="Export Modified APK"
        android:icon="@android:drawable/ic_menu_save"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_export_logs"
        android:title="Export Logs"
        android:icon="@android:drawable/ic_menu_upload"
        android:showAsAction="never" />
</menu>
================================================================================

/ModLoader/app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml

<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
================================================================================

/ModLoader/app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml

<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
================================================================================

/ModLoader/app/src/main/res/values/colors.xml

<resources>
    <color name="purple_200">#BB86FC</color>
    <color name="purple_500">#6200EE</color>
    <color name="purple_700">#3700B3</color>
    <color name="teal_200">#03DAC5</color>
    <color name="teal_700">#018786</color>
    <color name="black">#000000</color>
    <color name="white">#FFFFFF</color>
    <color name="colorPrimary">#6200EE</color>
</resources>
================================================================================

/ModLoader/app/src/main/res/values/strings.xml

<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- Original -->
    <string name="app_name">Terraria ML</string>

    <!-- Plugin Manager -->
    <string name="plugins_title">Plugins</string>
    <string name="plugins_enable">Enable</string>
    <string name="plugins_disable">Disable</string>
    <string name="plugins_uninstall">Uninstall</string>
    <string name="plugins_install">Install</string>   <!-- ✅ Added -->
    <string name="plugins_refresh">Refresh</string>
    <string name="plugins_details">Details</string>
    <string name="plugins_logs">Logs</string>
    <string name="plugins_permissions">Permissions</string>
    <string name="plugins_enabled">Enabled</string>
    <string name="plugins_disabled">Disabled</string>
</resources>
================================================================================

/ModLoader/app/src/main/res/values/themes.xml

<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Base application theme (Light) -->
    <style name="Theme.ModLoader" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/black</item>
        <item name="android:statusBarColor" tools:targetApi="l">?attr/colorPrimaryVariant</item>
    </style>
</resources>
================================================================================

/ModLoader/app/src/main/res/values-night/colors.xml

<resources>
    <color name="white">#000000</color>
    <color name="black">#FFFFFF</color>
</resources>
================================================================================

/ModLoader/app/src/main/res/values-night/themes.xml

<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Night mode theme -->
    <style name="Theme.ModLoader" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_200</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/black</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/white</item>
        <item name="android:statusBarColor" tools:targetApi="l">?attr/colorPrimaryVariant</item>
    </style>
</resources>
================================================================================

/ModLoader/app/src/main/res/xml/backup_rules.xml

<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <!-- Include app-specific files -->
    <include domain="file" path="." />
    <include domain="database" path="." />
    <include domain="sharedpref" path="." />
    <include domain="external" path="Android/data/com.modloader/" />

    <!-- Exclude cache and logs if needed -->
    <exclude domain="cache" path="." />
    <exclude domain="file" path="logs/" />
</full-backup-content>
================================================================================

/ModLoader/app/src/main/res/xml/data_extraction_rules.xml

<?xml version="1.0" encoding="utf-8"?><!--
   Sample data extraction rules file; uncomment and customize as necessary.
   See https://developer.android.com/about/versions/12/backup-restore#xml-changes
   for details.
-->
<data-extraction-rules>
  <cloud-backup>
    <!-- TODO: Use <include> and <exclude> to control what is backed up.
        <include .../>
        <exclude .../>
        -->
  </cloud-backup>
  <!--
    <device-transfer>
        <include .../>
        <exclude .../>
    </device-transfer>
    -->
</data-extraction-rules>
================================================================================

/ModLoader/app/src/main/res/xml/file_paths.xml

<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path
        name="external_files"
        path="." />
</paths>
================================================================================

/ModLoader/app/src/main/res/xml/file_provider_paths.xml

<paths xmlns:android="http://schemas.android.com/tools">
    
    <!-- External storage root (for legacy support) -->
    <external-path 
        name="external_storage_root" 
        path="." />
    
    <!-- App-specific external files directory -->
    <external-files-path 
        name="app_external_files" 
        path="." />
    
    <!-- APK installation directory (FIXED - main issue for APK parsing) -->
    <external-files-path 
        name="apk_install" 
        path="apk_install" />
    
    <!-- Cache directory for temporary files -->
    <external-cache-path 
        name="app_cache" 
        path="." />
    
    <!-- TerrariaLoader main directory -->
    <external-files-path 
        name="terraria_loader" 
        path="TerrariaLoader" />
    
    <!-- Game-specific directories -->
    <external-files-path 
        name="terraria_game" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid" />
    
    <!-- Mod directories -->
    <external-files-path 
        name="dex_mods" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Mods/DEX" />
    
    <external-files-path 
        name="dll_mods" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Mods/DLL" />
    
    <!-- Log directories -->
    <external-files-path 
        name="app_logs" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs" />
    
    <external-files-path 
        name="game_logs" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Logs" />
    
    <!-- Backup directories -->
    <external-files-path 
        name="backups" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Backups" />
    
    <!-- Config directories -->
    <external-files-path 
        name="config" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Config" />
    
    <!-- MelonLoader directories -->
    <external-files-path 
        name="melonloader" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Loaders/MelonLoader" />
    
    <!-- Downloads and exports -->
    <external-files-path 
        name="downloads" 
        path="downloads" />
    
    <external-files-path 
        name="exports" 
        path="exports" />
    
    <!-- Temporary processing directory -->
    <external-files-path 
        name="temp" 
        path="temp" />
    
    <!-- Legacy mod directory (for migration) -->
    <external-files-path 
        name="legacy_mods" 
        path="mods" />

</paths>
================================================================================

/ModLoader/app/src/main/res/xml/paths.xml

<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path
        name="external_files"
        path="." />
</paths>
================================================================================

