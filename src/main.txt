use egui::*;
use eframe::{App, Frame, NativeOptions};
use enigo::{Enigo, MouseControllable};
use rand::Rng;
use scrap::Display;
use std::{sync::{Arc, Mutex, atomic::{AtomicBool, Ordering}}, thread, time::Duration};

struct CursorMoverApp {
    minutes: String,
    status: Arc<Mutex<String>>, // Use Arc<Mutex<String>> for thread safety
    progress: Arc<Mutex<f32>>,
    running: Arc<Mutex<bool>>,
    reset_flag: Arc<Mutex<bool>>, // New field to signal reset
    stop_signal: Arc<AtomicBool>, // Signal to stop the thread
    elapsed_minutes: Arc<Mutex<u32>>, // Tracks elapsed minutes
    active_thread: Arc<Mutex<Option<thread::JoinHandle<()>>>>, // Handle for the active thread
}

impl Default for CursorMoverApp {
    fn default() -> Self {
        Self {
            minutes: String::new(),
            status: Arc::new(Mutex::new("Enter the number of minutes and press Start".to_string())),
            progress: Arc::new(Mutex::new(0.0)),
            running: Arc::new(Mutex::new(false)),
            reset_flag: Arc::new(Mutex::new(false)),
            stop_signal: Arc::new(AtomicBool::new(false)),
            elapsed_minutes: Arc::new(Mutex::new(0)),
            active_thread: Arc::new(Mutex::new(None)),
        }
    }
}

impl App for CursorMoverApp {
    fn update(&mut self, ctx: &Context, _frame: &mut Frame) {
        CentralPanel::default().show(ctx, |ui| {
            let available_height = ctx.screen_rect().height();
            let content_height = ui.spacing().item_spacing.y * 6.0 + 200.0; // Dynamic estimate of content height
            let vertical_padding = (available_height - content_height) / 2.0;

            // Top padding
            ui.add_space(vertical_padding);

            ui.vertical_centered(|ui| {
                ui.heading(RichText::new("Cursor Mover").strong().color(ctx.style().visuals.selection.bg_fill));
                ui.add_space(10.0);

                let is_running = *self.running.lock().unwrap();

                ui.label("Enter the number of minutes for moving the cursor:");
                ui.add_enabled_ui(!is_running, |ui| {
                    ui.text_edit_singleline(&mut self.minutes);
                });

                if *self.reset_flag.lock().unwrap() {
                    self.minutes.clear();
                    *self.reset_flag.lock().unwrap() = false;
                }

                if ui.add_enabled(!is_running && *self.status.lock().unwrap() != "Stopping program...", Button::new("Start")).clicked() {
                    let minutes: u32 = match self.minutes.trim().parse() {
                        Ok(num) if num > 0 && num < 481 => num,
                        _ => {
                            *self.status.lock().unwrap() = "Please enter a valid positive number between 1 and 480.".to_string();
                            return;
                        }
                    };


                    // Stop any existing thread before starting a new one
                    if let Some(handle) = self.active_thread.lock().unwrap().take() {
                        self.stop_signal.store(true, Ordering::Relaxed);
                        let _ = handle.join();
                    }

                    *self.status.lock().unwrap() = format!("Moving the cursor every 60 seconds for {} minutes.", minutes);
                    *self.running.lock().unwrap() = true;
                    *self.progress.lock().unwrap() = 0.0;
                    *self.elapsed_minutes.lock().unwrap() = 0;
                    self.stop_signal.store(false, Ordering::Relaxed);

                    let progress_clone = Arc::clone(&self.progress);
                    let running_clone = Arc::clone(&self.running);
                    let status_clone = Arc::clone(&self.status);
                    let reset_flag_clone = Arc::clone(&self.reset_flag);
                    let stop_signal_clone = Arc::clone(&self.stop_signal);
                    let elapsed_minutes_clone = Arc::clone(&self.elapsed_minutes);

                    let (screen_width, screen_height) = match Display::primary() {
                        Ok(d) => (d.width() as i32, d.height() as i32),
                        Err(e) => {
                            *self.status.lock().unwrap() = format!("Error: Failed to get the primary display. {e}");
                            *self.running.lock().unwrap() = false;
                            return;
                        }
                    };

                    let handle = thread::spawn(move || {
                        let mut enigo = Enigo::new();
                        for i in 0..minutes {
                            for _ in 0..600 { // Проверяем каждые 100 мс в течение одной минуты
                                if stop_signal_clone.load(Ordering::Relaxed) {
                                    *running_clone.lock().unwrap() = false;
                                    *status_clone.lock().unwrap() = "Program stopped.".to_string();
                                    *reset_flag_clone.lock().unwrap() = true;
                                    return;
                                }
                                thread::sleep(Duration::from_millis(100));
                            }

                            let (x, y) = (
                                rand::thread_rng().gen_range(0..screen_width),
                                rand::thread_rng().gen_range(0..screen_height),
                            );
                            enigo.mouse_move_to(x, y);

                            *progress_clone.lock().unwrap() = (i + 1) as f32 / minutes as f32;
                            *elapsed_minutes_clone.lock().unwrap() = i + 1;
                        }

                        *running_clone.lock().unwrap() = false;
                        *status_clone.lock().unwrap() = "Program stopped.".to_string();
                        *reset_flag_clone.lock().unwrap() = true;
                    });

                    *self.active_thread.lock().unwrap() = Some(handle);
                }

                if is_running {
                    let progress = *self.progress.lock().unwrap();
                    ui.add_sized([
                        ui.available_width() - 100.0, // Match field width
                        20.0
                    ], ProgressBar::new(progress).show_percentage());

                    if ui.button("Stop").clicked() {
                        self.stop_signal.store(true, Ordering::Relaxed);
                        *self.status.lock().unwrap() = "Stopping program...".to_string();

                        // Wait for the thread to stop without blocking UI
                        if let Some(handle) = self.active_thread.lock().unwrap().take() {
                            thread::spawn(move || {
                                let _ = handle.join();
                            });
                        }

                        *self.running.lock().unwrap() = false;
                        *self.reset_flag.lock().unwrap() = true;
                    }
                } else {
                    let status = self.status.lock().unwrap().clone();
                    ui.label(status);
                }
            });

            ui.add_space(vertical_padding); // Bottom padding
        });

        if *self.running.lock().unwrap() {
            // Display elapsed time in the bottom-right corner
            ctx.layer_painter(LayerId::new(Order::Foreground, Id::new("elapsed_time")))
                .text(
                    Pos2::new(ctx.screen_rect().max.x - 40.0, ctx.screen_rect().max.y - 30.0),
                    Align2::RIGHT_BOTTOM,
                    format!(
                        "{} / {} minutes",
                        *self.elapsed_minutes.lock().unwrap(),
                        self.minutes.parse::<u32>().unwrap_or(0)
                    ),
                    TextStyle::Body.resolve(&ctx.style()),
                    ctx.style().visuals.selection.bg_fill, // Added missing color argument
                );
        }

        ctx.request_repaint();
    }
}

fn main() -> Result<(), eframe::Error> {
    let app = CursorMoverApp::default();
    let native_options = NativeOptions {
        initial_window_size: Some(Vec2::new(400.0, 300.0)),
        ..Default::default()
    };
    eframe::run_native("Cursor Mover", native_options, Box::new(|_| Box::new(app)))
}
