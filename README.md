# 🏋️‍♂️ Weighty
A gym utility app that solves a specific, universally annoying problem: you enter the weights available to you and your target bar weight, and it calculates exactly which plates to load. Simple, functional, and built because the problem genuinely needed solving. Available on iOS and Android:

# 📱 Features
- Accounts for closest available weight when you set in available plates and bar weight.
- Clean visuals, clear UX design.
- Settings allows you to change accent colour.
- Sleek coding techniques for an efficient load.

<br>

<img width="281" height="621" alt="image" src="https://github.com/user-attachments/assets/ad7952c4-c687-4654-bad9-4bccdcdd7b82" />

<img width="278" height="622" alt="image" src="https://github.com/user-attachments/assets/5b9d1101-b038-4e09-8202-05edf2bdf212" />

<img width="276" height="623" alt="image" src="https://github.com/user-attachments/assets/299dc2b6-73c0-43b7-a66d-d4003f382e43" />

<img width="281" height="626" alt="image" src="https://github.com/user-attachments/assets/a3430f2a-ee11-4566-8222-75f672a00e02" />

<br>
<br>


```kotlin
package com.example.barbells

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.clickable
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Close
import androidx.compose.material.icons.filled.Remove
import androidx.compose.material.icons.filled.Settings
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.example.barbells.ui.theme.*

import androidx.compose.ui.tooling.preview.Preview

@Preview(showBackground = true, backgroundColor = 0xFF000000)
@Composable
fun BarbellCalculatorPreview() {
    BarBellsTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            BarbellCalculatorApp()
        }
    }
}

data class Plate(val weight: Double, val totalQuantity: Int)
data class UsedPlate(val weight: Double, val count: Int)

sealed class CalculationResult {
    object Idle : CalculationResult()
    data class Success(val used: List<UsedPlate>, val totalWeight: Double, val perSide: Double) : CalculationResult()
    data class Partial(val used: List<UsedPlate>, val remaining: Double, val achievedWeight: Double) : CalculationResult()
    data class Error(val message: String) : CalculationResult()
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            BarBellsTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    BarbellCalculatorApp()
                }
            }
        }
    }
}

@Composable
fun BarbellCalculatorApp() {
    var accentColor by remember { mutableStateOf(Accent) }
    var showColorPicker by remember { mutableStateOf(false) }

    BarBellsTheme(accentColor = accentColor) {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            var plates by remember {
                mutableStateOf(
                    listOf(
                        Plate(25.0, 4), Plate(20.0, 4), Plate(15.0, 2), Plate(10.0, 4),
                        Plate(5.0, 4), Plate(2.5, 4), Plate(1.25, 4), Plate(0.5, 2)
                    )
                )
            }

            var targetWeight by remember { mutableStateOf("") }
            var barWeight by remember { mutableStateOf(20.0) }
            var result by remember { mutableStateOf<CalculationResult>(CalculationResult.Idle) }

            Box(modifier = Modifier.fillMaxSize()) {
                LazyColumn(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(16.dp),
                    verticalArrangement = Arrangement.spacedBy(16.dp)
                ) {
                    item {
                        Header()
                    }

                    item {
                        SectionHeader("Available plates (total count)")
                    }

                    items(plates) { plate ->
                        PlateRow(
                            plate = plate,
                            onQtyChange = { delta ->
                                plates = plates.map {
                                    if (it.weight == plate.weight) it.copy(totalQuantity = (it.totalQuantity + delta).coerceAtLeast(1))
                                    else it
                                }
                            },
                            onRemove = {
                                plates = plates.filter { it.weight != plate.weight }
                            }
                        )
                    }

                    item {
                        AddPlateForm(onAdd = { w, q ->
                            plates = if (plates.any { it.weight == w }) {
                                plates.map { if (it.weight == w) it.copy(totalQuantity = q) else it }
                            } else {
                                (plates + Plate(w, q)).sortedByDescending { it.weight }
                            }
                        })
                    }

                    item { HorizontalDivider(color = MaterialTheme.colorScheme.tertiary, thickness = 0.5.dp) }

                    item {
                        TargetSection(
                            targetWeight = targetWeight,
                            onTargetWeightChange = { targetWeight = it },
                            barWeight = barWeight,
                            onBarWeightChange = { barWeight = it },
                            onCalculate = {
                                val target = targetWeight.toDoubleOrNull()
                                if (target == null || target <= 0) {
                                    result = CalculationResult.Error("Please enter a target weight.")
                                    return@TargetSection
                                }
                                if (target < barWeight) {
                                    result = CalculationResult.Error("Target must be at least $barWeight kg (bar weight).")
                                    return@TargetSection
                                }

                                val needed = (target - barWeight) / 2.0
                                var remaining = needed
                                val used = mutableListOf<UsedPlate>()

                                for (p in plates) {
                                    if (remaining <= 0.0001) break
                                    val availablePerSide = p.totalQuantity / 2
                                    val canUse = kotlin.math.min(availablePerSide, (remaining / p.weight + 0.0001).toInt())
                                    if (canUse > 0) {
                                        used.add(UsedPlate(p.weight, canUse))
                                        remaining -= canUse * p.weight
                                    }
                                }

                                val platesPerSide = used.sumOf { it.weight * it.count }
                                val actualTotal = barWeight + platesPerSide * 2

                                result = if (remaining > 0.001) {
                                    CalculationResult.Partial(used, remaining, actualTotal)
                                } else {
                                    CalculationResult.Success(used, actualTotal, platesPerSide)
                                }
                            }
                        )
                    }

                    item {
                        when (val r = result) {
                            is CalculationResult.Error -> ErrorBox(r.message)
                            is CalculationResult.Success -> ResultsView(r.used, r.totalWeight, r.perSide, barWeight)
                            is CalculationResult.Partial -> {
                                Column {
                                    ErrorBox("Cannot make exactly $targetWeight kg. Closest: ${r.achievedWeight} kg.")
                                    if (r.used.isNotEmpty()) {
                                        ResultsView(r.used, r.achievedWeight, (r.achievedWeight - barWeight) / 2.0, barWeight)
                                    }
                                }
                            }
                            CalculationResult.Idle -> {}
                        }
                    }

                    item { Spacer(modifier = Modifier.height(32.dp)) }
                }

                IconButton(
                    onClick = { showColorPicker = true },
                    modifier = Modifier
                        .align(Alignment.TopEnd)
                        .padding(16.dp)
                ) {
                    Icon(
                        Icons.Default.Settings,
                        contentDescription = "Settings",
                        tint = TextMuted
                    )
                }
            }
        }
    }

    if (showColorPicker) {
        AlertDialog(
            onDismissRequest = { showColorPicker = false },
            title = { Text("App Settings") },
            text = {
                Column {
                    Text("Select Accent Color", modifier = Modifier.padding(bottom = 16.dp))
                    Row(
                        horizontalArrangement = Arrangement.spacedBy(16.dp),
                        modifier = Modifier.padding(bottom = 16.dp)
                    ) {
                        ColorOption(color = Color(0xFFE8F44D), isSelected = accentColor == Color(0xFFE8F44D)) {
                            accentColor = Color(0xFFE8F44D)
                        }
                        ColorOption(color = Color(0xFFFF9500), isSelected = accentColor == Color(0xFFFF9500)) {
                            accentColor = Color(0xFFFF9500)
                        }
                        ColorOption(color = Color(0xFF54D6FF), isSelected = accentColor == Color(0xFF54D6FF)) {
                            accentColor = Color(0xFF54D6FF)
                        }
                    }
                    Row(
                        horizontalArrangement = Arrangement.spacedBy(16.dp),
                        modifier = Modifier.padding(bottom = 16.dp)
                    ) {
                        ColorOption(color = Color(0xFFF8BBD0), isSelected = accentColor == Color(0xFFF8BBD0)) {
                            accentColor = Color(0xFFF8BBD0)
                        }
                        ColorOption(color = Color(0xFFFFFFFF), isSelected = accentColor == Color(0xFFFFFFFF)) {
                            accentColor = Color(0xFFFFFFFF)
                        }
                        ColorOption(color = Color(0xFF9E9E9E), isSelected = accentColor == Color(0xFF9E9E9E)) {
                            accentColor = Color(0xFF9E9E9E)
                        }
                    }
                    Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
                        ColorOption(color = Color(0xFF4CAF50), isSelected = accentColor == Color(0xFF4CAF50)) {
                            accentColor = Color(0xFF4CAF50)
                        }
                        ColorOption(color = Color(0xFF795548), isSelected = accentColor == Color(0xFF795548)) {
                            accentColor = Color(0xFF795548)
                        }
                        ColorOption(color = Color(0xFF9C27B0), isSelected = accentColor == Color(0xFF9C27B0)) {
                            accentColor = Color(0xFF9C27B0)
                        }
                    }
                }
            },
            confirmButton = {
                TextButton(onClick = { showColorPicker = false }) {
                    Text("Close")
                }
            },
            containerColor = AppSurface,
            titleContentColor = TextPrimary,
            textContentColor = TextPrimary
        )
    }
}

@Composable
fun ColorOption(color: Color, isSelected: Boolean, onClick: () -> Unit) {
    Box(
        modifier = Modifier
            .size(48.dp)
            .clip(RoundedCornerShape(24.dp))
            .background(color)
            .border(
                width = if (isSelected) 3.dp else 0.dp,
                color = if (isSelected) Color.White else Color.Transparent,
                shape = RoundedCornerShape(24.dp)
            )
            .clickable { onClick() }
    )
}

@Composable
fun Header() {
    Column(modifier = Modifier.padding(vertical = 8.dp)) {
        Row(verticalAlignment = Alignment.Bottom) {
            Text(
                text = "W",
                style = MaterialTheme.typography.displayLarge,
                fontWeight = FontWeight.Black,
                fontSize = 48.sp,
                color = MaterialTheme.colorScheme.primary
            )
            Text(
                text = "EIGHTY",
                style = MaterialTheme.typography.displayLarge,
                fontWeight = FontWeight.Black,
                fontSize = 48.sp,
                color = TextPrimary
            )
        }
        Text(
            text = "BARBELL PLATE CALCULATOR",
            style = MaterialTheme.typography.labelSmall,
            color = TextMuted,
            letterSpacing = 2.sp
        )
    }
}

@Composable
fun SectionHeader(text: String) {
    Text(
        text = text.uppercase(),
        style = MaterialTheme.typography.labelSmall,
        color = TextMuted,
        fontWeight = FontWeight.Bold,
        letterSpacing = 1.5.sp,
        modifier = Modifier.padding(bottom = 8.dp)
    )
}

@Composable
fun PlateRow(plate: Plate, onQtyChange: (Int) -> Unit, onRemove: () -> Unit) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clip(RoundedCornerShape(10.dp))
            .background(AppSurface)
            .border(0.5.dp, Color.White.copy(alpha = 0.08f), RoundedCornerShape(10.dp))
            .padding(12.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Box(
            modifier = Modifier
                .size(12.dp)
                .clip(RoundedCornerShape(50))
                .background(getPlateColor(plate.weight))
        )
        Spacer(modifier = Modifier.width(12.dp))
        Text(text = "${plate.weight} kg", modifier = Modifier.weight(1f), fontWeight = FontWeight.Medium)
        
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier
                .clip(RoundedCornerShape(6.dp))
                .background(AppSurface2)
        ) {
            IconButton(onClick = { onQtyChange(-1) }, modifier = Modifier.size(32.dp)) {
                Icon(Icons.Default.Remove, contentDescription = "Decrease", modifier = Modifier.size(16.dp))
            }
            Box(
                modifier = Modifier
                    .width(40.dp)
                    .height(32.dp)
                    .background(AppSurface3),
                contentAlignment = Alignment.Center
            ) {
                Text(text = plate.totalQuantity.toString(), fontSize = 14.sp, fontWeight = FontWeight.Bold)
            }
            IconButton(onClick = { onQtyChange(1) }, modifier = Modifier.size(32.dp)) {
                Icon(Icons.Default.Add, contentDescription = "Increase", modifier = Modifier.size(16.dp))
            }
        }
        
        IconButton(onClick = onRemove) {
            Icon(Icons.Default.Close, contentDescription = "Remove", tint = TextMuted, modifier = Modifier.size(18.dp))
        }
    }
}

@Composable
fun AddPlateForm(onAdd: (Double, Int) -> Unit) {
    var weight by remember { mutableStateOf("") }
    var qty by remember { mutableStateOf("4") }

    Row(
        modifier = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        OutlinedTextField(
            value = weight,
            onValueChange = { weight = it },
            placeholder = { Text("Weight", fontSize = 14.sp) },
            modifier = Modifier.weight(1f),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Decimal),
            shape = RoundedCornerShape(10.dp),
            colors = OutlinedTextFieldDefaults.colors(
                focusedContainerColor = AppSurface,
                unfocusedContainerColor = AppSurface,
                focusedBorderColor = MaterialTheme.colorScheme.primary,
                unfocusedBorderColor = Color.White.copy(alpha = 0.08f)
            )
        )
        OutlinedTextField(
            value = qty,
            onValueChange = { qty = it },
            placeholder = { Text("Total Qty", fontSize = 14.sp) },
            modifier = Modifier.width(100.dp),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
            shape = RoundedCornerShape(10.dp),
            colors = OutlinedTextFieldDefaults.colors(
                focusedContainerColor = AppSurface,
                unfocusedContainerColor = AppSurface,
                focusedBorderColor = MaterialTheme.colorScheme.primary,
                unfocusedBorderColor = Color.White.copy(alpha = 0.08f)
            )
        )
        Button(
            onClick = {
                val w = weight.toDoubleOrNull()
                val q = qty.toIntOrNull()
                if (w != null && q != null && w > 0 && q > 0) {
                    onAdd(w, q)
                    weight = ""
                }
            },
            shape = RoundedCornerShape(10.dp),
            colors = ButtonDefaults.buttonColors(containerColor = AppSurface2)
        ) {
            Text("+ Add", color = TextPrimary)
        }
    }
}

@Composable
fun TargetSection(
    targetWeight: String,
    onTargetWeightChange: (String) -> Unit,
    barWeight: Double,
    onBarWeightChange: (Double) -> Unit,
    onCalculate: () -> Unit
) {
    val barOptions = listOf(20.0, 15.0, 10.0, 7.5, 0.0)
    val barLabels = mapOf(
        20.0 to "20 kg — Olympic",
        15.0 to "15 kg — Women's",
        10.0 to "10 kg — EZ Curl",
        7.5 to "7.5 kg — Fixed",
        0.0 to "0 kg — No Bar"
    )
    var expanded by remember { mutableStateOf(false) }

    Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
        SectionHeader("Target Lift")
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            OutlinedTextField(
                value = targetWeight,
                onValueChange = onTargetWeightChange,
                label = { Text("Target Weight (kg)", fontSize = 11.sp) },
                modifier = Modifier.weight(1f),
                keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Decimal),
                shape = RoundedCornerShape(10.dp),
                colors = OutlinedTextFieldDefaults.colors(
                    focusedBorderColor = MaterialTheme.colorScheme.primary,
                    unfocusedBorderColor = Color.White.copy(alpha = 0.08f)
                )
            )
            
            Box(modifier = Modifier.weight(1.2f)) {
                OutlinedTextField(
                    value = barLabels[barWeight] ?: "$barWeight kg",
                    onValueChange = {},
                    readOnly = true,
                    label = { Text("Bar Weight", fontSize = 11.sp) },
                    modifier = Modifier.fillMaxWidth(),
                    shape = RoundedCornerShape(10.dp),
                    trailingIcon = {
                        IconButton(onClick = { expanded = true }) {
                            Icon(Icons.Default.Add, contentDescription = "Select Bar", modifier = Modifier.size(16.dp))
                        }
                    },
                    colors = OutlinedTextFieldDefaults.colors(
                        focusedBorderColor = MaterialTheme.colorScheme.primary,
                        unfocusedBorderColor = Color.White.copy(alpha = 0.08f)
                    )
                )
                DropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
                    barOptions.forEach { weight ->
                        DropdownMenuItem(
                            text = { Text(barLabels[weight] ?: "$weight kg") },
                            onClick = {
                                onBarWeightChange(weight)
                                expanded = false
                            }
                        )
                    }
                }
            }
        }
        
        Button(
            onClick = onCalculate,
            modifier = Modifier.fillMaxWidth(),
            shape = RoundedCornerShape(10.dp),
            colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.primary)
        ) {
            Text("CALCULATE", color = Color.Black, fontWeight = FontWeight.Black, fontSize = 20.sp)
        }
    }
}

@Composable
fun ErrorBox(message: String) {
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .padding(top = 8.dp)
            .clip(RoundedCornerShape(10.dp))
            .background(ErrorColor.copy(alpha = 0.1f))
            .border(0.5.dp, ErrorColor.copy(alpha = 0.3f), RoundedCornerShape(10.dp))
            .padding(12.dp)
    ) {
        Text(text = message, color = ErrorColor, fontSize = 13.sp)
    }
}

@Composable
fun ResultsView(used: List<UsedPlate>, totalWeight: Double, perSide: Double, barWeight: Double) {
    Column(modifier = Modifier.padding(top = 16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            StatCard("Total Weight", "${if (totalWeight % 1 == 0.0) totalWeight.toInt() else totalWeight} kg", modifier = Modifier.weight(1f), isAccent = true)
            StatCard("Each Side", "${if (perSide % 1 == 0.0) perSide.toInt() else perSide} kg", modifier = Modifier.weight(1f))
            StatCard("Plates/Side", "${used.sumOf { it.count }} pcs", modifier = Modifier.weight(1f))
        }
        
        BarbellVisual(used, barWeight)
        
        BreakdownTable(used, barWeight, totalWeight, perSide)
    }
}

@Composable
fun StatCard(label: String, value: String, modifier: Modifier = Modifier, isAccent: Boolean = false) {
    val accentColor = MaterialTheme.colorScheme.primary
    Column(
        modifier = modifier
            .clip(RoundedCornerShape(10.dp))
            .background(if (isAccent) accentColor else AppSurface)
            .border(0.5.dp, if (isAccent) accentColor else Color.White.copy(alpha = 0.08f), RoundedCornerShape(10.dp))
            .padding(12.dp)
    ) {
        Text(text = label.uppercase(), fontSize = 9.sp, fontWeight = FontWeight.Bold, color = if (isAccent) Color.Black.copy(alpha = 0.6f) else TextMuted)
        Text(text = value, fontSize = 24.sp, fontWeight = FontWeight.Black, color = if (isAccent) Color.Black else TextPrimary)
    }
}

@Composable
fun BarbellVisual(used: List<UsedPlate>, barWeight: Double) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .clip(RoundedCornerShape(10.dp))
                .background(AppSurface)
                .padding(vertical = 24.dp, horizontal = 12.dp),
            contentAlignment = Alignment.Center
        ) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                // Left end
                Box(modifier = Modifier.size(10.dp, 20.dp).background(Color.Gray))
                
                // Left Plates
                Row(verticalAlignment = Alignment.CenterVertically) {
                    used.reversed().forEach { up ->
                        repeat(up.count) {
                            PlateBlock(up.weight)
                        }
                    }
                }
                
                // Bar Center
                Box(modifier = Modifier.width(60.dp).height(8.dp).background(Color.DarkGray))
                
                // Right Plates
                Row(verticalAlignment = Alignment.CenterVertically) {
                    used.forEach { up ->
                        repeat(up.count) {
                            PlateBlock(up.weight)
                        }
                    }
                }
                
                // Right end
                Box(modifier = Modifier.size(10.dp, 20.dp).background(Color.Gray))
            }
        }
        Text(text = "Symmetric load • $barWeight kg bar", fontSize = 10.sp, color = TextMuted, modifier = Modifier.padding(top = 8.dp))
    }
}

@Composable
fun PlateBlock(weight: Double) {
    val height = when {
        weight >= 25 -> 60.dp
        weight >= 20 -> 54.dp
        weight >= 15 -> 48.dp
        weight >= 10 -> 40.dp
        weight >= 5 -> 32.dp
        weight >= 2.5 -> 24.dp
        else -> 18.dp
    }
    val width = when {
        weight >= 25 -> 14.dp
        weight >= 20 -> 12.dp
        weight >= 15 -> 10.dp
        weight >= 10 -> 8.dp
        else -> 6.dp
    }
    Box(
        modifier = Modifier
            .padding(horizontal = 1.dp)
            .size(width, height)
            .clip(RoundedCornerShape(2.dp))
            .background(getPlateColor(weight))
    )
}

@Composable
fun BreakdownTable(used: List<UsedPlate>, barWeight: Double, totalWeight: Double, perSide: Double) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .clip(RoundedCornerShape(10.dp))
            .background(AppSurface)
            .border(0.5.dp, Color.White.copy(alpha = 0.08f), RoundedCornerShape(10.dp))
    ) {
        Row(
            modifier = Modifier.fillMaxWidth().padding(12.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Text("PLATE", fontSize = 10.sp, fontWeight = FontWeight.Bold, color = TextMuted)
            Text("PER SIDE", fontSize = 10.sp, fontWeight = FontWeight.Bold, color = TextMuted)
            Text("TOTAL", fontSize = 10.sp, fontWeight = FontWeight.Bold, color = TextMuted)
        }
        
        HorizontalDivider(color = Color.White.copy(alpha = 0.08f))
        
        BreakdownRow("Bar", "", "$barWeight kg", isMuted = true)
        
        used.forEach { up ->
            val sideW = up.weight * up.count
            val totalW = sideW * 2
            BreakdownRow(
                "${up.count} × ${up.weight} kg",
                "${if (sideW % 1 == 0.0) sideW.toInt() else sideW} kg/side",
                "${if (totalW % 1 == 0.0) totalW.toInt() else totalW} kg",
                color = getPlateColor(up.weight)
            )
        }
        
        Box(modifier = Modifier.background(AppSurface2)) {
            BreakdownRow(
                "Total",
                "${if (perSide % 1 == 0.0) perSide.toInt() else perSide} kg/side",
                "${if (totalWeight % 1 == 0.0) totalWeight.toInt() else totalWeight} kg",
                isBold = true
            )
        }
    }
}

@Composable
fun BreakdownRow(label: String, side: String, total: String, isMuted: Boolean = false, isBold: Boolean = false, color: Color? = null) {
    Row(
        modifier = Modifier.fillMaxWidth().padding(12.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment = Alignment.CenterVertically
    ) {
        Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.weight(1f)) {
            if (color != null) {
                Box(modifier = Modifier.size(8.dp).clip(RoundedCornerShape(50)).background(color))
                Spacer(modifier = Modifier.width(8.dp))
            }
            Text(label, fontSize = 14.sp, fontWeight = if (isBold) FontWeight.Bold else FontWeight.Normal, color = if (isMuted) TextMuted else TextPrimary)
        }
        Text(side, modifier = Modifier.weight(1f), fontSize = 13.sp, color = TextMuted, textAlign = androidx.compose.ui.text.style.TextAlign.End)
        Text(total, modifier = Modifier.weight(0.8f), fontSize = 14.sp, fontWeight = FontWeight.Bold, color = TextPrimary, textAlign = androidx.compose.ui.text.style.TextAlign.End)
    }
}
```
